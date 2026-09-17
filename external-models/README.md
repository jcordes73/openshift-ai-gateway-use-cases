# External Model

## Setting Environment Variables

The following environment variables need to be adapted to the environment, where the external model is hosted (the examples below are from an Demo System)

```bash
export EXTERNAL_PROVIDER_HOST=maas-rhdp.apps.maas.redhatworkshops.io
export EXTERNAL_MODEL=qwen3-14b
export EXTERNAL_MODEL_NAME=qwen3-14b
export EXTERNAL_MODEL_API_KEY=<api-key>
```

## Provision External Model Respurces

Now we are ready to provision the resouces in order to manage the external model:

```bash
cat external-model-provider-api-secret.yaml | envsubst | oc create -f -
cat external-model-provider.yaml | envsubst | oc create -f -
cat external-model.yaml | envsubst | oc create -f -
cat external-model-maas-ref.yaml | envsubst | oc create -f -
cat external-model-maas-subscription.yaml | envsubst | oc create -f -
cat external-model-maas-authpolicy.yaml | envsubst | oc create -f -
```

Get an **API-KEY** from OpenShift AI and store it and the API endpoint in an environment variable as we will need them later.

```bash
export OPENAI_API_KEY=$(
curl -sk -X POST \
  -H "Authorization: Bearer $(oc whoami -t)" \
  -H "Content-Type: application/json" \
  "https://maas.${OPENSHIFT_APPS_DOMAIN}/maas-api/v1/api-keys" \
  -d "{\"name\": \"${EXTERNAL_MODEL}\", \"description\": \"Key for ${EXTERNAL_MODEL}\", \"expiresIn\": \"30d\", \"subscription\": \"${EXTERNAL_MODEL}\"}" | jq -r '.key')
  
export OPENAI_API_ENDPOINT=https://maas.${OPENSHIFT_APPS_DOMAIN}/llm-hosting/${EXTERNAL_MODEL}
```

## Testing External Model Access

### Qwen Code

To test using [Qwen Code](https://qwen.ai/qwencode) execute
```bash
export QWEN_CODE_MAX_OUTPUT_TOKENS=8192

qwen --openai-base-url="${OPENAI_API_ENDPOINT}/v1" --openai-api-key="${OPENAI_API_KEY}" --model="${EXTERNAL_MODEL_NAME}" --insecure
```

### CURL
A simple test using curl looks like this
```bash
curl -k -X POST -H "Authorization: Bearer ${OPENAI_API_KEY}" -H "Content-Type: application/json" "${OPENAI_API_ENDPOINT}/v1/chat/completions" -s -d "$(cat example-message.json | envsubst)"
```