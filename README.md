# AI Gateway Use-Cases

This repository contains use-cases for demonstrating the capabilities of [Red Hat OpenShift AI](https://www.redhat.com/en/products/ai/openshift-ai) and related products when it comes to AI Gateway functionalities.

## Prerequisites

- **OpenShift Client** (oc)
- **OpenShift AI 3.5** (installed with the [OpenShift AI Ansible playbook](https://github.com/jcordes/openshift-ai-ansible))

## Preparation

1. Login into the OpenShift environment where OpenShift AI 3.5 is installed as **cluster-admin**
    ```bash
    oc login -u <cluster-admin> -p <password> <API URL>
    ```

2. Set the **OpenShift Apps Domain** environment variable:

    ```bash
    export OPENSHIFT_APPS_DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
    ```


## Use-Cases

You can find instructions for the the following use-cases here:

- [**Authentication/Authorization**](authentication/README.md)
- Managing [**External Models**](external-models/README.md)
- Protecting models with [**Guardrails**](guardrails/README.md)
- Deploying and managing [**MCP Servers**](mcp-servers/README.md)
- Deploying and managing [**Agents**](agents/README.md)

