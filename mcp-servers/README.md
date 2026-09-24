# MCP Servers

## Get the MCP Gateway URL

For most of the operations in this section we will need the **MCP_URL**:
```bash
export MCP_HOSTNAME="mcp.${OPENSHIFT_APPS_DOMAIN}"
export MCP_URL="https://${MCP_HOSTNAME}/mcp"
```

## Deploy Insights MCP Server
Get service account credentials from [console.redhat.com](https://console.redhat.com/iam/service-accounts) and set environbment variables
```bash
export INSIGHTS_CLIENT_ID=<INSIGHTS_CLIENT_ID>
export INSIGHTS_CLIENT_SECRET=<INSIGHTS_CLIENT_SECRET>
```
Now we can created the **Insights MCP Server**:
```bash
cat insights/insights-credentials-secret.yaml | envsubst | oc create -f -
oc create -f insights/insights-mcp-server.yaml
oc create -f insights/insights-mcp-server-httproute.yaml
oc create -f insights/insights-mcp-server-registration.yaml
```

## Deploy the OpenShift MCP Server

To deploy the **OpenShift MCP Server** run the following commands
```bash
oc create -f openshift/openshift-mcp-server-sa.yaml
oc create -f openshift/openshift-mcp-server.yaml 
oc create -f openshift/openshift-mcp-server-httproute.yaml 
oc create -f openshift/openshift-mcp-server-registration.yaml
```

## Verification

Get the Session ID for the MCP Broker
```bash
curl -ks -D mcp_headers "${MCP_URL}" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json,text/event-stream" \
  -H "Authorization: Bearer ${OPENSHIFT_ACCESS_TOKEN}" \
  -d '{"jsonrpc": "2.0", "id": 1, "method": "initialize", "params": {"protocolVersion": "2025-11-25", "capabilities": {}, "clientInfo": {"name": "test-client", "version": "1.0.0"}}}'
  
SESSION_ID=$(grep -i "mcp-session-id:" mcp_headers | cut -d' ' -f2 | tr -d '\r')
```

To get the available tools use the following call
```bash
curl -sk "${MCP_URL}" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "mcp-session-id: ${SESSION_ID}" \
  -H "Authorization: Bearer $(oc whoami -t)" \
  -d '{"jsonrpc": "2.0", "id": 2, "method": "tools/list"}'
```
For getting also the available prompts use this call:
```bash
curl -sk "${MCP_URL}" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "mcp-session-id: ${SESSION_ID}"  \
  -H "Authorization: Bearer $(oc whoami -t)" \
  -d '{"jsonrpc": "2.0", "id": 2, "method": "prompts/list"}'
```
The tools and prompts belonging to a particular MCP server are prefixed with the ```prefix``` that was specified inside the ```MCPServerRegistration```CR.

> ℹ️ 
> Please note that there are two tools in the output that are specific to the MCP Gateway itself:
> * discover_tools
> * select_tools
 
## Authentication

For authentication per MCP server a client needs to be defined in the ```clients``` section section of the ```KeycloakRealmImport``` CR, where the convention ```<namespace>/<mcp-server-name>``` is used as the ID of the client.

There is also a special client for the MCP gateway itself.

```yaml
clients:
  - clientId: mcp-gateway
    name: MCP Gateway Client
    enabled: true
    publicClient: true
    directAccessGrantsEnabled: true
    standardFlowEnabled: true
    implicitFlowEnabled: false
    redirectUris:
      - "http://localhost:*"
      - "http://127.0.0.1:*"
      - "https://localhost:*"
    webOrigins:
      - "*"
    protocol: openid-connect
    attributes:
      pkce.code.challenge.method: S256
      client.secret.creation.time: "0"
    defaultClientScopes:
      - openid
      - roles
      - groups
  - clientId: llm-hosting/insights-mcp-server
    name: Insights MCP Server Resource
    enabled: true
    bearerOnly: true
    publicClient: false
    serviceAccountsEnabled: false
    protocol: openid-connect
    defaultClientScopes:
      - openid
```

The available tools need to be listed in the ```clientScopeMappings``` section of the ```KeycloakRealmImport``` CR, where the convention ```<namespace>/<mcp-server-name>``` is used to reference the Keycloak client, below you can find how this looks like for the Insights and OpenShift MCP servers:
```yaml
clientScopeMappings:
  llm-hosting/insights-mcp-server:
    - client: mcp-gateway
      roles:
        - "tool:insights_mcp_server_advisor__get_active_rules"
        - "tool:insights_mcp_server_advisor__get_hosts_details_for_rule"
        - "tool:insights_mcp_server_advisor__get_hosts_hitting_a_rule"
        - "tool:insights_mcp_server_advisor__get_recommendations_stats"
        - "tool:insights_mcp_server_advisor__get_rule_by_text_search"
        - "tool:insights_mcp_server_advisor__get_rule_details"
        - "tool:insights_mcp_server_advisor__get_rule_from_node_id"
        - "tool:insights_mcp_server_content-sources__list_repositories"
        - "tool:insights_mcp_server_get_mcp_version"
        - "tool:insights_mcp_server_image-builder__get_blueprint_details"
        - "tool:insights_mcp_server_image-builder__get_blueprints"
        - "tool:insights_mcp_server_image-builder__get_compose_details"
        - "tool:insights_mcp_server_image-builder__get_composes"
        - "tool:insights_mcp_server_image-builder__get_distributions"
        - "tool:insights_mcp_server_image-builder__get_openapi"
        - "tool:insights_mcp_server_image-builder__get_org_id"
        - "tool:insights_mcp_server_inventory__find_host_by_name"
        - "tool:insights_mcp_server_inventory__get_host_details"
        - "tool:insights_mcp_server_inventory__get_host_system_profile"
        - "tool:insights_mcp_server_inventory__get_host_tags"
        - "tool:insights_mcp_server_inventory__get_workspace"
        - "tool:insights_mcp_server_inventory__list_hosts"
        - "tool:insights_mcp_server_inventory__list_workspace_hosts"
        - "tool:insights_mcp_server_inventory__list_workspaces"
        - "tool:insights_mcp_server_inventory__load_inventory_dashboard"
        - "tool:insights_mcp_server_planning__get_appstreams_lifecycle"
        - "tool:insights_mcp_server_planning__get_relevant_appstreams"
        - "tool:insights_mcp_server_planning__get_relevant_rhel_lifecycle"
        - "tool:insights_mcp_server_planning__get_relevant_upcoming"
        - "tool:insights_mcp_server_planning__get_rhel_lifecycle"
        - "tool:insights_mcp_server_planning__get_upcoming_changes"
        - "tool:insights_mcp_server_rbac__get_all_access"
        - "tool:insights_mcp_server_rhsm__get_activation_key"
        - "tool:insights_mcp_server_rhsm__get_activation_keys"
        - "tool:insights_mcp_server_vulnerability__explain_cves"
        - "tool:insights_mcp_server_vulnerability__get_cve"
        - "tool:insights_mcp_server_vulnerability__get_cve_systems"
        - "tool:insights_mcp_server_vulnerability__get_cves"
        - "tool:insights_mcp_server_vulnerability__get_openapi"
        - "tool:insights_mcp_server_vulnerability__get_system_cves"
        - "tool:insights_mcp_server_vulnerability__get_systems"
        - "tool:insights_mcp_server_vulnerability__load_cve_dashboard"
```

To enable OAUth authentication we need to deploy a ```Kuadrant``` CR, patch the existing ```MCPGatewayExtension``` and create an ```AuthPolicy``` for the **MCP Gateway**.
```bash
oc create -f mcp-gateway-kuadrant.yaml

oc patch mcpgatewayextension mcp-extension -n mcp-gateway --type=merge -p "{
  \"spec\": {
    \"oauthProtectedResource\": {
      \"resourceName\": \"MCP Gateway\",
      \"resource\": \"${MCP_URL}\",
      \"authorizationServers\": [\"${KEYCLOAK_ISSUER}\"],
      \"bearerMethodsSupported\": [\"header\"],
      \"scopesSupported\": [\"openid\", \"groups\", \"roles\"]
    }
  }
}"

cat mcp-gateway-auth-policy.yaml | envsubst | oc create -f -
```
Then we create the ```AuthPolicy```for each MCP Server:
```bash
cat insights/insights-mcp-server-authpolicy.yaml | envsubst | oc create -f -
cat openshift/openshift-mcp-server-authpolicy.yaml | envsubst | oc create -f -
```

To verify authentication use these calls:
```bash
TOKEN=$(curl -sk "${KEYCLOAK_ISSUER}/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password" \
  -d "client_id=mcp-gateway" \
  -d "username=operations-user" \
  -d "password=operations" \
  -d "scope=openid groups roles" | jq -r '.access_token')

curl -sk -D mcp_headers "${MCP_URL}" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "Authorization: Bearer ${TOKEN}" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2025-06-18",
      "capabilities": {},
      "clientInfo": {"name": "test-client", "version": "1.0.0"}
    }
  }'
  ```
Now you can also list all tools available
```bash
MCP_SESSION_ID=$(grep -i "mcp-session-id:" mcp_headers | cut -d' ' -f2 | tr -d '\r')

curl -sk "${MCP_URL}"   -H "Content-Type: application/json"   -H "Accept: application/json, text/event-stream"   -H "Authorization: Bearer ${TOKEN}"   -H "mcp-session-id: ${MCP_SESSION_ID}"   -d '{"jsonrpc": "2.0", "id": 2, "method": "tools/list"}' | jq -r '.result.tools[].name'
```

## Authorization

Now it is time to implement authorization in addition to authentication.
```bash
cat mcp-gateway-authz-policy.yaml | envsubst | oc create -f -
```

In **Keycloak** there are three different groups:

- development
- operations
- security

Each of those groups has access to a different set of tools provided by the MCP servers.

### Unauthorized

To test authentication we will authenticate as a user belonging to the **development** group. This group doesn't have access to the **rbac__get_caller_access** tool provided by the Insights MCP server, access is only available for members of the **security** group. 
```bash
TOKEN=$(curl -sk "${KEYCLOAK_ISSUER}/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password" \
  -d "client_id=mcp-gateway" \
  -d "username=developer-user" \
  -d "password=developer" \
  -d "scope=openid groups roles" | jq -r '.access_token')

curl -sk -D mcp_headers -X POST "${MCP_URL}" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${TOKEN}" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"test","version":"1.0.0"}}}'

MCP_SESSION_ID=$(grep -i "mcp-session-id:" mcp_headers | cut -d' ' -f2 | tr -d '\r')

curl -sk "${MCP_URL}" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "mcp-session-id: ${MCP_SESSION_ID}" \
  -d '{"jsonrpc": "2.0", "id": 2, "method": "tools/list"}'

curl -sk "${MCP_URL}" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "mcp-session-id: ${MCP_SESSION_ID}" \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"insights_mcp_server_rbac__get_caller_access"}}'
```

### Authorized

Now we will test with a member of the **security** group.

```bash
TOKEN=$(curl -sk "${KEYCLOAK_ISSUER}/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password" \
  -d "client_id=mcp-gateway" \
  -d "username=security-user" \
  -d "password=security" \
  -d "scope=openid groups roles" | jq -r '.access_token')

curl -sk -D mcp_headers "${MCP_URL}" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${TOKEN}" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"test","version":"1.0.0"}}}'

MCP_SESSION_ID=$(grep -i "mcp-session-id:" mcp_headers | cut -d' ' -f2 | tr -d '\r')

curl -sk "${MCP_URL}" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "mcp-session-id: ${MCP_SESSION_ID}" \
  -d '{"jsonrpc": "2.0", "id": 2, "method": "tools/list"}' | jq

curl -sk "${MCP_URL}" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "mcp-session-id: ${MCP_SESSION_ID}" \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"insights_mcp_server_rbac__get_caller_access"}}' \
  | | yq -o json '.data'
```

## Registration of Tools in Apicurio Registry

TODO This section needs to be revisited once entire MCP Servers can be registered in **Apicurio Registry**.

```bash
TOKEN=$(curl -sk "${KEYCLOAK_ISSUER}/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password" \
  -d "client_id=mcp-gateway" \
  -d "username=security-user" \
  -d "password=security" \
  -d "scope=openid groups roles" | jq -r '.access_token')


curl -sk -X POST "https://registry.${OPENSHIFT_APPS_DOMAIN}/apis/registry/v3/groups/mcp-tools/artifacts" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${TOKEN}" \
  -d '{
    "artifactId": "insights",
    "artifactType": "MCP_TOOL",
    "firstVersion": {
      "version": "1.0.0",
      "content": {
        "contentType": "application/json",
        "content": "${TOOLS}"
      }
    }
  }'
```