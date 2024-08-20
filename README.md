# Kubernetes Cluster Configuration with Minikube for GitHub Actions Runners

## Overview

This document outlines the steps to configure a Kubernetes cluster using Minikube on an AWS instance, set up a GitHub Actions Runner Controller, and implement horizontal scaling for dynamic runner management. The setup leverages Helm for package management and Kubernetes for orchestration, ensuring a scalable and reliable environment for CI/CD workflows.

## Prerequisites

- **AWS Instance**: Ensure you have an AWS instance with Ubuntu.
- **Root Access**: Administrative privileges are required for software installation and configuration.
- **GitHub Token**: A GitHub token with appropriate permissions to authenticate and manage runners.

## Steps

### 1. Install Minikube

Minikube is used to create a local Kubernetes cluster on your AWS instance.

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb
```

### 2. Install kubectl
kubectl is the Kubernetes command-line tool for interacting with your cluster.

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubectl
```

### 3. Install Docker
Docker is required for containerization, which is essential for running Kubernetes pods.

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker ${USER}
sudo service docker start
```

### 4. Start Minikube
Initialize your Kubernetes cluster with Minikube.

```bash
minikube start
```
### 5. Create a Namespace
Namespaces in Kubernetes allow you to divide cluster resources between multiple users.

```bash
kubectl create namespace actions-runner-system
```
### 6. Install cert-manager
cert-manager automates the management and issuance of TLS certificates.

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.2/cert-manager.yaml
```

### 7. Install Helm
Helm is a package manager for Kubernetes, simplifying the deployment of applications and services.

```bash
sudo snap install helm --classic
```

### 8. Add the Actions Runner Controller Helm Repository and Install the Controller
The Actions Runner Controller enables the use of self-hosted GitHub Actions runners within Kubernetes.

```bash
helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller

helm upgrade --install --namespace actions-runner-system --create-namespace\
  --set=authSecret.create=true\
  --set=authSecret.github_token="REPLACE_YOUR_TOKEN_HERE"\
  --wait actions-runner-controller actions-runner-controller/actions-runner-controller
```

### 9. Configure the Runner Deployment
Create a runnerdeployment.yaml file to define your runner deployment.

```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: example-runner-deployment
  namespace: actions-runner-system
spec:
  template:
    spec:
      repository: Cadaservices/hng_boilerplate_nextjs
      serviceAccountName: github-actions-runner
      labels:
        - self-hosted
        - linux
        - x64
```

Apply the configuration:

```bash
kubectl apply -f runnerdeployment.yaml
```

### 10. Create a Service Account and Set Permissions
A service account allows the GitHub Actions Runner to interact with the Kubernetes API.

```bash
kubectl create serviceaccount -n actions-runner-system github-actions-runner
kubectl create clusterrolebinding github-actions-runner --clusterrole=cluster-admin --serviceaccount=actions-runner-system:github-actions-runner
```

### 11. Assign Roles to the Service Account
Define the roles and permissions required for the GitHub Actions Runner.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: actions-runner-system
  name: github-actions-runner-role
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log", "secrets"]
  verbs: ["get", "list", "watch", "create", "delete", "update"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "delete", "update"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: github-actions-runner-rolebinding
  namespace: actions-runner-system
subjects:
- kind: ServiceAccount
  name: github-actions-runner
  namespace: actions-runner-system
roleRef:
  kind: Role
  name: github-actions-runner-role
  apiGroup: rbac.authorization.k8s.io
```

Apply the roles:
```bash
kubectl apply -f github-actions-runner-rbac.yaml
```

### 12. Configure Horizontal Scaling
To ensure your runners scale automatically based on workflow demand, create a horizontalscaler.yaml file:

```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: example-runner-deployment-autoscaler
  namespace: actions-runner-system
spec:
  scaleTargetRef:
    kind: RunnerDeployment
    name: example-runner-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: TotalNumberOfQueuedAndInProgressWorkflowRuns
    repositoryNames:
    - Cadaservices/hng_boilerplate_nextjs
```
Apply the configuration:

```bash
kubectl apply -f horizontalscaler.yaml
```

### 13. Verify the Setup
After applying all configurations, verify the deployment and scaling by checking the status of your pods and runner deployments:

```bash
kubectl get pods -n actions-runner-system
kubectl get runnerdeployments -n actions-runner-system
kubectl get horizontalrunnerautoscalers -n actions-runner-system
```
