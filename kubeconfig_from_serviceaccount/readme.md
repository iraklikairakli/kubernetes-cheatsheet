# Kubernetes QA Member Kubeconfig Setup

This guide walks you through the process of creating a service account with restricted access to view and list pods in the `default` namespace, generating a kubeconfig file for the QA member, and testing the configuration.

## Prerequisites

- **Kubernetes Cluster**: Ensure you have access to a Kubernetes cluster.
- **kubectl**: Installed and configured to interact with your Kubernetes cluster.

## Steps

### 1. Create the Service Account

First, create a service account named `qa-member` in the `default` namespace:

```bash
kubectl create serviceaccount qa-member -n default
```

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: qa-viewer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

```bash
kubectl create rolebinding qa-viewer-binding \
  --role=qa-viewer \
  --serviceaccount=default:qa-member \
  --namespace=default
```

```bash

TOKEN=$(kubectl create token qa-member --namespace default)
CLUSTER_NAME=$(kubectl config view -o jsonpath='{.contexts[?(@.name == "'$(kubectl config current-context)'")].context.cluster}')
CLUSTER_SERVER=$(kubectl config view -o jsonpath='{.clusters[?(@.name == "'$CLUSTER_NAME'")].cluster.server}')
CA_DATA=$(kubectl config view --raw -o jsonpath='{.clusters[?(@.name == "'$CLUSTER_NAME'")].cluster.certificate-authority-data}')
```


```bash
cat <<EOF > qa-member-kubeconfig.yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: $CA_DATA
    server: $CLUSTER_SERVER
  name: $CLUSTER_NAME
contexts:
- context:
    cluster: $CLUSTER_NAME
    namespace: default
    user: qa-member
  name: qa-member-context
current-context: qa-member-context
users:
- name: qa-member
  user:
    token: $TOKEN
EOF
```

```bash
kubectl --kubeconfig=qa-member-kubeconfig.yaml get pods
```
