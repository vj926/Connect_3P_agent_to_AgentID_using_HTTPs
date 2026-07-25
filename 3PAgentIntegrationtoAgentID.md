# Connecting a 3rd-Party Agent (Base44) to Microsoft Entra Agent Identity

**Platform:** Base44 (3rd-party agent platform) or any 3P platform  
**Identity Provider:** Microsoft Entra ID  

---

## Executive Summary

This document demonstrates how any **3rd-party agent (Base44 Agent in this session)** can authenticate using **Microsoft Entra Agent Identity**. 

Before proceeding with the below steps, check if Base44 Has Native OIDC Support. You need to check whether Base44 exposes an OIDC issuer URL or platform token for agents. If Base44 had native OIDC support, we could configure a Federated Identity Credential that trusts Base44's OIDC issuer directly — tokens would auto-rotate with no manual intervention. As of March 2026, Base44 does not have native OIDC support. This meant we needed the **manual FIC token path** — generate a FIC token externally and store it as a secret in Base44.

The integration is based entirely on:
- OAuth 2.0 `client_credentials`
- Federated Identity Credentials (FIC)
- JWT token exchange

This proves:

- Entra Agent Identity is **platform-agnostic**
- Any system capable of HTTP calls can integrate

---

## 1. Objective

Enable a **Base44 agent** to authenticate as a **Microsoft Entra Agent Identity** and obtain a valid Entra-issued JWT.

---

## 2. Pre-requisites

| Asset | Details |
|------|--------|
| Tenant ID | `Microsoft_Tenant_ID` |
| Blueprint | Steps in AgentID+Blueprint.md |
| Agent Identity | Steps in AgentID+Blueprint.md |
| Base44 Agent | Created |
| FIC Tokens | Steps in AgentID+Blueprint.md |

---

## 3. Architecture (High-Level Flow)
Blueprint App (client_id + secret)
↓
Generate FIC Token (JWT)
↓
Pass FIC as client_assertion
↓
Exchange for Agent Identity Token
↓
Use token in 3rd-party agent (Base44)


---

## 4. Core Concept

The flow works because Entra supports:

- OAuth2 token exchange
- JWT bearer assertions
- Federated credentials

---

## 5. End-to-End Flow

### Step 1 — Generate FIC Token (Blueprint)
In this step, the **Blueprint application authenticates using its own credentials** and requests a **FIC token for a specific Agent Identity**.

```bash
POST https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token
Authorization: Basic <BASE64(BLUEPRINT_CLIENT_ID:BLUEPRINT_CLIENT_SECRET)>
Content-Type: application/x-www-form-urlencoded
scope=api://AzureADTokenExchange/.default
grant_type=client_credentials
fmi_path=<AGENT_IDENTITY_APP_ID>
```
### Step 2 — Exchange FIC for Agent Identity Token
In this step, the **Agent Identity takes over**. The Blueprint already generated a **FIC token (JWT)** in Step 1. Now that token is used as proof to say: “I am allowed to act as this Agent Identity.”
```bash
POST https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded
client_id=<AGENT_IDENTITY_APP_ID>
grant_type=client_credentials
client_assertion=<FIC_TOKEN>
scope=https://graph.microsoft.com/.default
```
### Step 4: Deploy the FIC Token in Base44
In this step, we save the fresh FIC token as Secret in Base44. The Base44 backend function reads this secret at runtime to perform the token exchange. Storing it as a secret ensures it's encrypted and not exposed in code.
```bash
In the Base44 dashboard:
1. Open the agent's project settings
2. Navigated to Secrets / Environment Variables
3. Add a new secret:
   - **Key:** `MICROSOFT_AGENT_FIC_TOKEN`
   - **Value:** The FIC token from `fic-token.txt`
4. Save the token
```

### Step 5: Deploy Backend Function in Base44
In this step, we **connect Base44 with Microsoft Entra** by writing a backend function that performs the token exchange at runtime. 
Created a backend function `getEntraData` in Base44:

```typescript
const TENANT_ID = "<TENANT_ID>";
const AGENT_IDENTITY_APP_ID = "<AGENT_IDENTITY_APP_ID>";

async function exchangeFicTokenForAccessToken(ficToken) {
  const tokenEndpoint =
    `https://login.microsoftonline.com/${TENANT_ID}/oauth2/v2.0/token`;

  const body = new URLSearchParams({
    client_id: AGENT_IDENTITY_APP_ID,
    scope: "api://AzureADTokenExchange/.default",
    grant_type: "client_credentials",
    client_assertion_type:
      "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
    client_assertion: ficToken,
  });

  const response = await fetch(tokenEndpoint, {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: body.toString(),
  });

  if (!response.ok) {
    const errorText = await response.text();
    throw new Error(
      `Token exchange failed (${response.status}): ${errorText}`
    );
  }

  const data = await response.json();
  return data.access_token;
}

export default async function handler(req) {
  try {
    const ficToken = Deno.env.get("MICROSOFT_AGENT_FIC_TOKEN");
    if (!ficToken) {
      return new Response(
        JSON.stringify({
          error: "MICROSOFT_AGENT_FIC_TOKEN secret not configured",
        }),
        { status: 500, headers: { "Content-Type": "application/json" } }
      );
    }

    const accessToken = await exchangeFicTokenForAccessToken(ficToken);

    // Decode token payload to show identity
    const payloadB64 = accessToken.split(".")[1];
    const payload = JSON.parse(
      atob(payloadB64.replace(/-/g, "+").replace(/_/g, "/"))
    );

    return new Response(
      JSON.stringify({
        success: true,
        message:
          "Base44 Agent successfully authenticated as Entra Agent Identity",
        agentIdentity: {
          appId: AGENT_IDENTITY_APP_ID,
          subject: payload.sub,
          issuer: payload.iss,
          tenantId: payload.tid,
          tokenType: payload.idtyp,
          expiresAt: new Date(payload.exp * 1000).toISOString(),
        },
      }),
      { status: 200, headers: { "Content-Type": "application/json" } }
    );
  } catch (error) {
    const message = error instanceof Error ? error.message : String(error);
    return new Response(JSON.stringify({ error: message }), {
      status: 500,
      headers: { "Content-Type": "application/json" },
    });
  }
}
```
### Step 6:Trigger the `getEntraData` backend function from the Base44 agent.

**Result:**
```json
{
  "success": true,
  "message": "Base44 Agent successfully authenticated as Entra Agent Identity",
  "agentIdentity": {
    "appId": "<AGENT_IDENTITY_APP_ID>",
    "subject": "<AGENT_IDENTITY_APP_ID>",
    "issuer": "https://login.microsoftonline.com/<TENANT_ID>/v2.0",
    "tenantId": "<TENANT_ID>",
    "tokenType": "app",
    "expiresAt": "2026-03-26T08:48:36.000Z"
  }
}
```
