# The steps I applied in configuring a kubernates cluster using minikube

1. Install minikube

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb
```

2. Install kubectl
```bash
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
# Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL:

# If the folder `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg # allow unprivileged APT programs to read this keyring

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list   # helps tools such as command-not-found to work correctly

sudo apt-get update
sudo apt-get install -y kubectl
```

3. install docker and start docker

- Setup docker

```bash
Set up Docker's apt repository.


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

- install docker

```bash
 sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
- add user to docker group and start docker
```bash
sudo usermod -aG docker ${USER}
sudo service docker start
```

4. Start minikube

```bash
minikube start
```

5. Create namespace:

```bash
kubectl create namespace actions-runner-system
```

6. Install cert-manager in your cluster. For more information, see "cert-manager."

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.2/cert-manager.yaml
```

6. Install Helm which will help us install controllers

```bash
sudo snap install helm --classic
```

7. Add repository and install helm chart
```bash
helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller

helm upgrade --install --namespace actions-runner-system --create-namespace\
  --set=authSecret.create=true\
  --set=authSecret.github_token="REPLACE_YOUR_TOKEN_HERE"\
  --wait actions-runner-controller actions-runner-controller/actions-runner-controller
```

8. Create runnerdeployment.yaml file and add this code to it.

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

9. Apply file to the kubernates cluster

```bash
kubectl apply -f runnerdeployment.yaml
```

10. Create service account and set permission

```bash
kubectl create serviceaccount -n actions-runner-system github-actions-runner
kubectl create clusterrolebinding github-actions-runner --clusterrole=cluster-admin --serviceaccount=actions-runner-system:github-actions-runner
```

11. Set roles for service account

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

12. Apply roles to service account

```bash
kubectl apply -f github-actions-runner-rbac.yaml
```

13. Configure horizontal scaling

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

14. Apply configuration

```bash
kubectl apply -f horizontalscaler.yaml
```