Kubernetes QA Member Kubeconfig Setup
This guide walks you through the process of creating a service account with restricted access to view and list pods in the default namespace, generating a kubeconfig file for the QA member, and testing the configuration.

Prerequisites
Kubernetes Cluster: Ensure you have access to a Kubernetes cluster.
kubectl: Installed and configured to interact with your Kubernetes cluster.
Steps
1. Create the Service Account
First, create a service account named qa-member in the default namespace:

bash
Copy code
kubectl create serviceaccount qa-member -n default
2. Define and Apply the Role
Next, define a role that grants permissions to list and get pods and view logs in the default namespace. Save the following content to a file named qa-viewer-role.yaml:

yaml
Copy code
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
Apply the role using the following command:

bash
Copy code
kubectl apply -f qa-viewer-role.yaml
3. Create the Role Binding
Bind the qa-viewer role to the qa-member service account:

bash
Copy code
kubectl create rolebinding qa-viewer-binding \
  --role=qa-viewer \
  --serviceaccount=default:qa-member \
  --namespace=default
4. Generate the Kubeconfig File
Generate a token for the qa-member service account and gather necessary cluster information:

bash
Copy code
TOKEN=$(kubectl create token qa-member --namespace default)
CLUSTER_NAME=$(kubectl config view -o jsonpath='{.contexts[?(@.name == "'$(kubectl config current-context)'")].context.cluster}')
CLUSTER_SERVER=$(kubectl config view -o jsonpath='{.clusters[?(@.name == "'$CLUSTER_NAME'")].cluster.server}')
CA_DATA=$(kubectl config view --raw -o jsonpath='{.clusters[?(@.name == "'$CLUSTER_NAME'")].cluster.certificate-authority-data}')
Create the kubeconfig file for the qa-member service account:

bash
Copy code
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
5. Test the Kubeconfig
To ensure the kubeconfig is correctly set up, use the following command to test access to the default namespace:

bash
Copy code
kubectl --kubeconfig=qa-member-kubeconfig.yaml get pods
This command should list all pods in the default namespace, confirming that the qa-member service account has the correct permissions.

Conclusion
You have successfully created a kubeconfig file for a QA member with restricted access to the Kubernetes cluster. This configuration allows the QA member to view and list pods and view logs in the default namespace without the ability to modify or delete any resources.

