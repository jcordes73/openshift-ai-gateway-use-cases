# Authentication

For authentication and authorization we will be using [**Red Hat build of Keycloak**](https://access.redhat.com/products/red-hat-build-keycloak/) to protect the following aspects of Red Hat OpenShift AI.

- Red Hat OpenShift AI itself
- Models-as-a-Service
- MCP Gateway
- OGX

**Red Hat build of Keycloak** has already been installed as part of [OpenShift AI Ansible playbook](https://github.com/jcordes/openshift-ai-ansible)) in the **redhat-ods-applications** namespace.

## Setup

First we need to create a secret for the OpenShift Console Client:
```bash
export OPENSHIFT_CONSOLE_CLIENT_SECRET=$(openssl rand -base64 32)
```

To import the Realm definition into Keycloak run
```bash
cat keycloak-rhoai-import.yaml | envsubst | oc create -f -
```
For subsequent steps in other sections we need the ```KEYCLOAK_URL``` and the ```KEYCLOAK_ISSUER``` environment variables:
```bash
export KEYCLOAK_URL="https://keycloak-rhoai.${OPENSHIFT_APPS_DOMAIN}"
export KEYCLOAK_ISSUER="${KEYCLOAK_URL}/realms/rhoai"
```

Also in case there are any issues save the Keycloak Admin password:
```bash
export KEYCLOAK_ADMIN_PASSWORD=$(oc get secret keycloak-rhoai-initial-admin -n redhat-ods-applications -o json | jq -r '.data.password' | base64 -d)
```

Finally we have to create the secret that will be used by for authentication and the **Authentication** resource itself.

> ℹ️
> Please note that note that this will also switch the authentication for the OpenShift cluster itself and it will take several minutes to take effect.

As a safety net, create a service account with cluster-admin role and create a long-lived token:
```bash
oc create sa backup -n openshift-config
oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:openshift-config:backup

BACKUP_TOKEN=$(oc create token backup --duration 10080m -n openshift-config)
```

```bash
oc adm policy add-cluster-role-to-group cluster-admin cluster-admins
cat openshift-console-client-secret.yaml | envsubst | oc create -f -
cat openshift-authentication.yaml | envsubst | oc replace -f -
```
To login via the OpenShift CLI (oc) you have to use the following command
```bash
OPENSHIFT_BASE_DOMAIN=$(echo ${OPENSHIFT_APPS_DOMAIN} | cut -d'.' -f2-)
oc login https://api.${OPENSHIFT_BASE_DOMAIN}:6443 \
  --exec-plugin=oc-oidc \
  --issuer-url=${KEYCLOAK_ISSUER} \
  --client-id=openshift-console \
  --client-secret="${OPENSHIFT_CONSOLE_CLIENT_SECRET}" \
  --extra-scopes=email,profile \
  --callback-port=8080
  ```

Now we need to change the default **Gateway** as well
```bash
cat gateway-client-secret.yaml | envsubst | oc create -f -
oc patch gatewayconfig default-gateway --type='merge' -p='{
"spec": {
    "oidc": {
    "issuerURL": "'${KEYCLOAK_ISSUER}'",
    "clientID": "'openshift-console'",
    "clientSecretRef": {
        "name": "idp-client-secret",
        "key": "clientSecret"
    }
    }
}
}'
```

Workaround for

oc delete secret kube-auth-proxy-creds -n openshift-ingress
oc rollout restart depoc rollout restart deployment/kube-auth-proxy -n openshift-ingress

oc create -f rhoai-projects-cluster-role.yaml
oc adm policy add-cluster-role-to-group rhoai-projects-read rhods-admins
oc adm policy add-cluster-role-to-group self-provisioner rhods-admins

OPENSHIFT_ACCESS_TOKEN=$(curl -sk "${KEYCLOAK_ISSUER}/protocol/openid-connect/token" -d "grant_type=password"   -d "client_id=openshift-console" --data-urlencode "client_secret=${OPENSHIFT_CONSOLE_CLIENT_SECRET}"  -d "username=admin"   -d "password=openshift" | jq -r '.access_token')