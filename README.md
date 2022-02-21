# ArgoCrossplaneIntegration

##  What Is Argo CD?
- Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes.
- Argo CD follows the GitOps pattern of using Git repositories as the source of truth for defining the desired application state
- Argo CD automates the deployment of the desired application states in the specified target environments. Application deployments can track updates to branches, tags, or pinned to a specific version of manifests at a Git commit.


##  What Is Crossplane?
- Crossplane is an open source Kubernetes add-on that transforms your cluster into a universal control plane. Crossplane enables platform teams to assemble infrastructure from multiple vendors, and expose higher level self-service APIs for application teams to consume, without having to write any code.
- Crossplane extends your Kubernetes cluster to support orchestrating any infrastructure or managed service. Compose Crossplane’s granular resources into higher level abstractions that can be versioned, managed, deployed and consumed using your favorite tools and existing processes.

## prerequisite

- Setup Kubernetes Cluster
- Install Kubectl and Helm

### Install Crossplane using helm chart
```bash
helm repo add crossplane-stable \
    https://charts.crossplane.io/stable

helm repo update

helm upgrade --install \
    crossplane crossplane-stable/crossplane \
    --namespace crossplane-system \
    --create-namespace \
    --wait
```

### Install Crossplane CLI
```bash
curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh
sudo mv kubectl-crossplane /usr/bin
kubectl crossplane --help
```

### Give permission to crossplane to create resources inside google cloud

```bash
export PROJECT_ID=cybage-devops	
export SA_NAME=crossplane-sa
export SA="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

gcloud iam service-accounts \
    create $SA_NAME \
    --project $PROJECT_ID

export ROLE=roles/admin

gcloud projects add-iam-policy-binding \
    --role $ROLE $PROJECT_ID \
    --member serviceAccount:$SA

gcloud iam service-accounts keys \
    create creds.json \
    --project $PROJECT_ID \
    --iam-account $SA

kubectl --namespace crossplane-system \
    create secret generic gcp-creds \
    --from-file key=./creds.json
```

### Set provider as GCP
```bash
kubectl crossplane install provider \
    crossplane/provider-gcp:v0.15.0
    
kubectl get providers
```

### Create provider congiguration file and pass same secret that we created above to access gcp

```bash
echo "apiVersion: gcp.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  projectID: $PROJECT_ID
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: gcp-creds
      key: key" \
    | kubectl apply --filename -
```

### Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.0.4/manifests/install.yaml
```

### Install ArgoCD CLI
```bash
sudo curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.0.4/argocd-linux-amd64
sudo chmod +x /usr/local/bin/argocd
```

### Access The Argo CD API Server
```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

### Login Using The CLI
- retrive the initial admin password
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

- Login with server IP
```bash
argocd login <ARGOCD_SERVER>
```
- Change the password using the command
```bash
argocd account update-password
```
