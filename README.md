# Agent Identity Setup and Operations
This repository demonstrates the following activities related to **Microsoft Entra Agent Identity**.
## Activities Covered
1. Create an **Agent Identity Blueprint**
2. Create an **Agent Identity Blueprint Principal**
3. Add **Password Credential (Client Secret)** to the Agent Blueprint
4. Get **Access Token** from Agent Blueprint
5. **Expose API (scope)** for the Agent Blueprint
6. Create **Agent Identity**
7. Create **Agentic User**
8. Grent (consent) permissions for the Agentic user
9. Obtain a Federated Identity Credential (FIC) for the Agent Blueprint
10. Obtain a Federated Identity Credential (FIC) for the Agent Identity
11. Obtain an Agentic User Token for MS Graph using the agent_blueprint_fic-token and Agent_ID_fic_token
12. User the agent_user_token to call MS Graph/me endpoint
13. Get the groups the Agentic User (Digital colleague) is member of
14. Obtain the Agent Blueprint FIC Token (Autonomous Agent)
15. Obtain the Agent Identity MS Graph access token
---
# Environment Setup
## Required Tools
1. **Insomnia**  
   Download: https://insomnia.rest/download  
   *If your machine uses Windows OS, ensure Windows Defender allows external applications to open.*
2. **Microsoft Entra Admin Center**  
   Login to the Entra Admin Center.  
   You will need the following from this portal:
   - **Tenant ID**
   - **Sponsor ID**
3. Permissions needed for the excercise -
   **AgentID Administrator (From Entra Admin Center Portal)**
   **DelegatedPermissionGrant.ReadWrite.All (Microsoft Graph Explorer)**
---
# Setup Steps
## Create Environment variable mentioned below
1. Tenant ID - Go to Microsoft Entra Admin Center -> Overview Page -> Copy the Tenant ID
2. ACCESS_TOKEN - Microsoft Graph Explorer with Entra credentials -> Locate the Access Token on the screen -> Copy the Access Token
3. agent_sponsor_URI - Open Users (MS Entra Admin Center) -> Click on the Sponsor User (Can be any user who controls the Agent) -> Copy the Object ID of the user 
4. agent_blueprint_appId - Copy the App ID result from 01.01
5. client_secret - Copy the output "Secret Text" from 01.03
6. Agentaccess_token - Copy the result from 01.04
7. agent_user_upn - Create a User UPN with Text@tenantaddress. Example user@example.onmicrosoft.com
8. agent_user_mailNickName - You can create a Nickname for the Agent User Mail ID. Example vijAgent
9. GENERATED_GUID - randomly create a GUID from the internet
10. agent_identity_clientId - Execute 02.01 -> Entra Admin Center -> Agent ID (Preview) -> Click the Agent ID just created and copy the Client ID
11. agent_blueprint_ficToken - Execute 03.02 and copy the token from the output
12. agent_identity_ficToken - Execute 03.03 and copt the token from the output
13. agent_blueprint_appObjectId -Microsoft Entra Admin Center -> Agent ID (Preview) -> View All Agent Identities -> Agent Blueprints -> Click the Blueprint 
14. agent_user_accessToken -
15. token_url - https://login.microsoftonline.com/{{ _.Tenant_ID }}/oauth2/v2.0/token
---
Note - Remove directory.readwrite.all permission (if needed)
If the access token fails due to permission conflicts, remove the directory.readwrite.all permission.
reference documentation:  
[how to remove some of the permissions from graph explorer](https://learn.microsoft.com/en-us/answers/questions/1346583/how-to-remove-some-of-the-permissions-from-graph-e)

---
# Agent Identity End-to-End Lab Guide
<img width="1440" height="1561" alt="image" src="https://github.com/user-attachments/assets/ca142659-bd20-42c2-a103-fecc7fe2b5db" />

## 01 Creating the Agent ID Blueprint
### 01.01 Create the Agent Identity Blueprint (application)
When you create the Agent Identity Blueprint, you are only creating the definition or template of the agent identity. It describes what the agent is, its name, and who sponsors it. However, this blueprint itself cannot authenticate or access resources. The Blueprint Principal is the actual identity created in the tenant from that blueprint. This principal is what can sign in, obtain tokens, and be granted permissions. Microsoft keeps these two separate because the blueprint acts like a design, while the principal is the real identity created from that design. 
```bash
 curl --request POST \
  --url https://graph.microsoft.com/beta/applications/ \
  --header 'Authorization: Bearer {{ ACCESS_TOKEN }}' \
  --header 'Content-Type: application/json' \
  --header 'OData-Version: 4.0' \
  --header 'User-Agent: insomnia/12.4.0' \
  --data '{
  "@odata.type": "Microsoft.Graph.AgentIdentityBlueprint",
  "displayName": "[ai] Blueprint for Digital Worker 01",
  "sponsors@odata.bind": [
    "https://graph.microsoft.com/beta/users/{Agent_Sponsor_URI}"
  ]
}'
```
Expected result
```json
{
	"@odata.context": "https://graph.microsoft.com/beta/$metadata#applications/$entity",
	"@odata.type": "#microsoft.graph.agentIdentityBlueprint",
	"id": "<AGENT_BLUEPRINT_APP_ID>",
	"deletedDateTime": null,
	"appId": "<AGENT_BLUEPRINT_APP_ID>",
	"applicationTemplateId": null,
	"identifierUris": [],
	"createdByAppId": "<CREATED_BY_APP_ID>",
	"createdDateTime": "2026-03-12T20:37:33.8446502Z",
	"description": null,
	"disabledByMicrosoftStatus": null,
	"displayName": "[ai] Blueprint for Digital Worker 01",
	"isAuthorizationServiceEnabled": false,
	"isDeviceOnlyAuthSupported": null,
	"isDisabled": null,
	"isFallbackPublicClient": null,
	"isManagementRestricted": null,
	"groupMembershipClaims": null,
	"nativeAuthenticationApisEnabled": null,
	"notes": null,
	"oauth2RequirePostResponse": false,
	"orgRestrictions": [],
	"publisherDomain": "<PUBLISHER_DOMAIN>",
	"signInAudience": "AzureADMyOrg",
	"tags": [],
}
```
### 01.02 Create Agent Identity Blueprint Principal
Blueprint (Application) = Defines the identity
Blueprint Principal (Service Principal) = Actual identity that runs inside the tenant
```bash
curl --request POST \
  --url https://graph.microsoft.com/beta/serviceprincipals/graph.agentIdentityBlueprintPrincipal \
  --header 'Authorization: Bearer {{ ACCESS_TOKEN }} ' \
  --header 'Content-Type: application/json' \
  --header 'OData-Version: 4.0' \
  --header 'User-Agent: insomnia/12.4.0' \
  --data '{
  "appId": " Agent_blueprint_appId"
}'
```
Expected result
```json
{
	"error": {
		"code": "Request_MultipleObjectsWithSameKeyValue",
		"message": "The service principal cannot be created, updated, or restored because the service principal name <AGENT_BLUEPRINT_OBJECT_ID> is already in use.",
		"details": [
			{
				"code": "ObjectConflict",
				"message": "The service principal cannot be created, updated, or restored because the service principal name <AGENT_BLUEPRINT_OBJECT_ID> is already in use.",
				"target": "None",
				"blockedWord": "",
				"prefix": "",
				"suffix": ""
			}
		],
		"innerError": {
			"date": "2026-03-12T23:50:07",
			"request-id": "<REQUEST_ID>",
			"client-request-id": "<REQUEST_ID>"
		}
	}
}
```
### 01.03  Add Password credential (client secret) to the Agent Blueprint
A password credential (client secret) is added to the Agent Blueprint Application that was created in 01.01. This credential functions as a secret that the application can use during authentication. Although the blueprint is a template, the actual identity created from the Blueprint operating within the tenant (the blueprint principal or service principal) requires a method to prove its identity when requesting a token from Entra ID. The password credential stored on the application object is used for this purpose. When the agent requests an access token, Entra ID verifies the Application ID and the associated secret before issuing the token. In summary, the earlier steps established the agent identity, and this step provides the credential required for that identity to authenticate and obtain tokens.
```bash
curl --request POST \
  --url https://graph.microsoft.com/beta/applications/Agent_Blueprint_ObjectID/addPassword \
  --header 'Authorization: Bearer {{ ACCESS_TOKEN }}' \
  --header 'Content-Type: application/json' \
  --header 'User-Agent: insomnia/12.4.0' \
  --data '{
  "passwordCredential": {
    "displayName": "Dummy Secret",
    "endDateTime": "2026-08-05T23:59:59Z"
  }
}'
```
Expected Respone
```json
{
	"@odata.context": "https://graph.microsoft.com/beta/$metadata#microsoft.graph.passwordCredential",
	"customKeyIdentifier": null,
	"endDateTime": "2026-08-05T23:59:59Z",
	"keyId": "<KEY_ID>",
	"startDateTime": "2026-03-12T23:17:19.5583321Z",
	"secretText": "<CLIENT_SECRET>",
	"hint": "<SECRET_HINT>",
	"displayName": "Dummy Secret"
}
```
### 01.04  Get Access Token from Agent Blueprint
Access Tokens are used for the Authorization (proving that the app has permission to access a resource). In Step 01.04, the Agent Blueprint application that was created in Step 01.01 uses the Application ID from that step and the password credential (client secret) that was added in Step 01.03 to authenticate with Entra ID’s token endpoint. During this request, Entra ID verifies that the Application ID and the client secret are valid. If the verification succeeds, Entra ID issues an access token. This access token is then used in later API requests to prove that the application has permission to access resources such as Microsoft Graph.
```bash
curl --request POST \
  --url https://login.microsoftonline.com/TENANT_ID/oauth2/v2.0/token \
  --header 'Authorization: Bearer YOUR_BLUENPRINT_APP_ID' \
  --header 'Content-Type: application/x-www-form-urlencoded' \
  --header 'OData-Version: 4.0' \
  --header 'User-Agent: insomnia/12.4.0' \
  --data client_id=YOUR_BLUEPRINT_APP_ID \
  --data 'client_secret=YOUR_SECRET_FROM_01.03_OUTPUT' \
  --data scope=https://graph.microsoft.com/.default \
  --data grant_type=client_credentials
```
Expected response
```json
{
	"token_type": "Bearer",
	"expires_in": 3599,
	"ext_expires_in": 3599,
	"access_token": "ey***"
}
```
### 01.05  Exposing API (scope) for the Agent Blueprint
In Step 01.05, we are modifying the Agent Blueprint application that was created in Step 01.01 to expose an API permission (scope). This request uses the access token obtained in Step 01.04 to call Microsoft Graph and update the application configuration. In this step, the application is given an API identifier (api://{{ _.agent_blueprint_appId }}) and a permission scope called access_agent. This means that the Agent Blueprint application now exposes a permission that other applications can request when they want to interact with the agent.
```bash
curl --request PATCH \
  --url https://graph.microsoft.com/beta/applications/{{ _.agent_blueprint_appObjectId }} \
  --header 'Authorization: Bearer {{ ACCESS_TOKEN }}' \
  --header 'Content-Type: application/json' \
  --header 'User-Agent: insomnia/12.4.0' \
  --data '{
  "identifierUris": [
    "api://{{ _.agent_blueprint_appId }}"
  ],
  "api": {
    "oauth2PermissionScopes": [
      {
        "adminConsentDescription": "Allow the application to access the agent on behalf of the signed-in user.",
        "adminConsentDisplayName": "Access agent",
        "id": "{{ GENERATED_GUID }}",
        "isEnabled": true,
        "type": "User",
        "value": "access_agent"
      }
    ]
  }
}'
```
## 02 Creating the Agent Identity
In Step 02.01, you are creating the actual Agent ID by using the Agent Blueprint application created in Step 01.01. The key link to the earlier steps is the field agentAppId, which points to the Application ID of the Agent Blueprint. This tells Microsoft Graph which blueprint should be used as the base for creating this agent identity.
### 02.01  Creating Agent ID
```bash
curl --request POST \
  --url https://graph.microsoft.com/beta/serviceprincipals/Microsoft.Graph.AgentIdentity \
  --header 'Authorization: Bearer {{ ACCESS_TOKEN }}' \
  --header 'Content-Type: application/json' \
  --header 'OData-Version: 4.0' \
  --header 'User-Agent: insomnia/12.4.0' \
  --data '{
    "displayName": "[ai] Agent Identity for Digital Worker 01",
    "agentAppId": "{{ _.agent_blueprint_appId }}",
	  "sponsors@odata.bind": [
      "https://graph.microsoft.com/beta/users/{{ _.agent_sponsor_URI }}"
    ]
  }'
```
Expected Response
```json
{
	"@odata.context": "https://graph.microsoft.com/beta/$metadata#servicePrincipals/microsoft.graph.agentIdentity/$entity",
	"id": "<AGENT_IDENTITY_APP_ID>",
	"deletedDateTime": null,
	"accountEnabled": true,
	"alternativeNames": [],
	"createdByAppId": "<AGENT_BLUEPRINT_OBJECT_ID>",
	"createdDateTime": null,
	"deviceManagementAppType": null,
	"appDescription": null,
	"appDisplayName": null,
	"appId": "<AGENT_IDENTITY_APP_ID>",
	"applicationTemplateId": null,
	"appOwnerOrganizationId": null,
	"appRoleAssignmentRequired": false,
	"assignmentRequiredForPrincipalTypes": null,
	"description": null,
	"disabledByMicrosoftStatus": null,
	"displayName": "[ai] Agent Identity for Digital Worker 01",
	....
}
```
### 02.02  Creating Agentic UserID
This step creates an Agent User and associates it with the Agent ID created in Step 02.01. The relationship with the earlier steps is established through the identityParentId field. This field contains the client ID of the Agent ID, thereby linking the newly created Agent User to that specific Agent ID. In the earlier sequence, Step 01.01 created the Agent Blueprint application, and Step 02.01 used that blueprint to create the Agent ID. In this step, the Agent User is created as a child identity under that Agent ID by referencing its client ID through identityParentId.
```bash
curl --request POST \
  --url https://graph.microsoft.com/beta/users \
  --header 'Authorization: Bearer {{ ACCESS_TOKEN_FROM_OAUTH2_CLIENT_CREDENTIALS }}' \
  --header 'Content-Type: application/json' \
  --header 'OData-Version: 4.0' \
  --header 'User-Agent: insomnia/12.4.0' \
  --data '{
    "@odata.type":"microsoft.graph.agentUser",
    "displayName": "[ai] Digital Worker 01 Agent User",
    "userPrincipalName": "AGENT_USER_UPN",
    "mailNickname": "AGENT_USER_NICKNAME",
    "accountEnabled": true,
    "identityParentId":"AGENT_IDENTITY_CLIENTID"
  }'
```
### 02.03  Grant (consent) permissions for the Agentic User
You need Tenant_ID, agent_identity_clientId to enable the permissions. We provide consent explicitly to the Agentic user. Add the below link in the browser and you are redirected to Entra to consent with the permissions. 
```bash
https://login.microsoftonline.com/{{ _.tenant_id }}/v2.0/adminconsent?client_id={{agent_identity_clientId}}&scope=User.Read+groupmember.read.all+Chat.ReadWrite+Calendars.ReadWrite+Mail.ReadWrite+Contacts.Read+People.Read&redirect_uri=https://entra.microsoft.com/TokenAuthorize&state=xyz123
```
## 03 Authenticate Agentic User
### 03.01  Obtain a Federated Identity Credetial (FIC) for the Agent Blueprint
In this flow, the Agent Blueprint application (created in Step 01.01) authenticates using the client secret added in Step 01.03. However, the actual operations in the system should be performed by the Agent ID created in Step 02.01, because that identity represents the digital worker or agent. The token exchange process allows the system to take the authentication performed by the Agent Blueprint application and obtain a token that represents the Agent ID instead. This ensures that when the agent accesses resources or performs actions, the system records the activity under the Agent ID, not under the blueprint application.
```bash
curl --request POST \
  --url _.token_url \
  --header 'Authorization: Basic {{ _.agent_blueprint_appId }}:{{ _.agent_blueprint_clientSecret }}' \
  --header 'Content-Type: application/x-www-form-urlencoded' \
  --header 'User-Agent: insomnia/12.4.0' \
  --data scope=api://AzureADTokenExchange/.default \
  --data grant_type=client_credentials \
  --data fmi_path={{ _.agent_identity_clientId }}
```
### 03.02  Obtain a Federated Identity Credetial (FIC) for the Agent Identity
Only Blueprint application can authenticate directly with Entra ID. Because of this design, the system first authenticates the Agent Blueprint application and obtains a FIC token in Step 03.02. That token is then used in Step 03.03 as a client assertion to obtain a token for the Agent Identity. This process is called token exchange, where the authenticated blueprint application requests a token on behalf of the agent identity. FIC token is specifically required for the token exchange process that allows the blueprint application to obtain a token representing the Agent Identity.
```bash
curl --request POST \
  --url token_url \
  --header 'Content-Type: application/x-www-form-urlencoded' \
  --header 'User-Agent: insomnia/12.4.0' \
  --data client_id={{ _.agent_identity_clientId }} \
  --data scope=api://AzureADTokenExchange/.default \
  --data grant_type=client_credentials \
  --data client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer \
  --data client_assertion= {{ _.agent_blueprint_ficToken }}
```
### 03.03  Obtain an Agentic User token for MS Graph using the agent_blueprint_fic-token and agent_id_fic-token
Step 01.01 created the Agent Blueprint application, Step 02.01 created the Agent Identity from that blueprint, the later user step created the Agent User under that Agent Identity, Step 03.01 obtained the Blueprint FIC token, Step 03.02 obtained the Agent Identity FIC token, and this step combines those pieces to request a token for accessing Microsoft Graph on behalf of that Agent User.
```bash
curl --request POST \
  --url token_url \
  --header 'Content-Type: multipart/form-data' \
  --header 'User-Agent: insomnia/12.4.0' \
  --form client_id=agent_identity_clientId \
  --form client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer \
  --form client_assertion={{ _.agent_blueprint_ficToken }} \
  --form grant_type=user_fic \
  --form requested_token_use=on_behalf_of \
  --form scope=https://graph.microsoft.com/.default \
  --form username={{ _.agent_user_upn }} \
  --form user_federated_identity_credential= {{ _.agent_identity_ficToken }}
```
### 03.04  Use the agent-user-token to call MS Graph /me endpoint
This step uses the access token obtained in the previous step (the token generated after the on-behalf-of flow) to call Microsoft Graph. The request calls the /me endpoint, which returns information about the currently authenticated user. In this case, the Bearer token (agent_user_accessToken) represents the Agent User that was created earlier and whose identity was used in the on-behalf-of request.
```bash
curl --request GET \
  --url https://graph.microsoft.com/v1.0/me \
  --header 'Authorization: Bearer {{ _.agent_user_accessToken }}' \
  --header 'User-Agent: insomnia/12.4.0'
```
### 03.05 Get the groups the Agentic User (Digital Colleague) is member of
This step uses the Agent User access token obtained in the previous step to call Microsoft Graph and retrieve the security groups that the Agent User belongs to. The request calls the /me/getMemberGroups endpoint, which returns the IDs of all security-enabled groups associated with the currently authenticated user.
```bash
curl --request POST \
  --url https://graph.microsoft.com/v1.0/me/getMemberGroups \
  --header 'Authorization: Bearer {{ _.agent_user_accessToken }}' \
  --header 'Content-Type: application/json' \
  --header 'User-Agent: insomnia/12.4.0' \
  --data '{
  "securityEnabledOnly": true
}'
```
## 04 Authenticate the Agent Identity (Autonomous Agent)
### 04.01 Obtain the Agent Blueprint FIC token
There is no fundamental difference in purpose between this code and the earlier Blueprint FIC token code. Both are requesting a Federated Identity Credential token for the Agent Blueprint, and in both cases the request is tied to the Agent Identity through fmi_path={{ _.agent_identity_clientId }}. The main difference is only in how the request is written. In the earlier version, the parameters were sent using --data with application/x-www-form-urlencoded. In this version, they are sent using --form with multipart/form-data. So the format of the HTTP request is different, but the logical purpose is the same. (How do we know if this autonomous or not ?? And why is the process different for this versus earlier one)
```bash
curl --request POST \
  --url {{ _.token_url }}\
  --header 'Authorization: Basic {{ _.agent_blueprint_appId }}:{{ _.client_secret }}' \
  --header 'Content-Type: multipart/form-data' \
  --header 'User-Agent: insomnia/12.4.0' \
  --form scope=api://AzureADTokenExchange/.default \
  --form grant_type=client_credentials \
  --form fmi_path={{ _.agent_identity_clientId }}
```
### 04.02 Obtain the Agent Identity MS Graph acess token
```bash
curl --request POST \
  --url {{ _.token_url }} \
  --header 'Content-Type: application/x-www-form-urlencoded' \
  --header 'User-Agent: insomnia/12.4.0' \
  --data client_id={{ _.agent_identity_clientId }} \
  --data scope=https://graph.microsoft.com/.default \
  --data grant_type=client_credentials \
  --data client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer \
  --data client_assertion={{ _.agent_blueprint_ficToken }}
```
