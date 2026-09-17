# Agents

For deploying and managing agents an **OGX** (formerly Llama-Stack) environment is needed.

## Setting up an OGX Server

A **OGX server** is bound to a particular LLM.

```bash
cat ogx-server-config.yaml | envsubst | oc create -f -
cat ogx-server.yaml | envsubst | oc create -f -
```


curl -X POST localhost:8321/v1/toolgroups \
-H "Content-Type: application/json" \
--data \
'{ "provider_id" : "model-context-protocol", "toolgroup_id" : "mcp::filesystem", "mcp_endpoint" : { "uri" : "http://localhost:8002/sse"}}'