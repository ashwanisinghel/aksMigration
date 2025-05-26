
# Deploying C# Console App to AKS using GitHub Actions

## 📌 Prerequisites
1. Azure CLI installed or access to Azure Portal.
2. AKS Cluster already created and accessible.
3. Dockerfile in your C# project root.
4. GitHub repository with `dev` and `main` branches.
5. Azure Container Registry (ACR) or Docker Hub.
6. Kubernetes manifests (`deployment.yaml`, `service.yaml`, etc.).
7. GitHub Secrets set for automation.

---

## 🔧 Step 1: Prepare Your Project

### ✅ Add a Dockerfile

```dockerfile
FROM mcr.microsoft.com/dotnet/runtime:8.0 AS base
WORKDIR /app

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "YourAppName.dll"]
```

---

## 📁 Step 2: Add Kubernetes YAMLs

### ✅ `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: csharp-console-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: csharp-console-app
  template:
    metadata:
      labels:
        app: csharp-console-app
    spec:
      containers:
      - name: csharp-console-app
        image: <ACR_OR_DOCKERHUB>/csharp-console-app:latest
        imagePullPolicy: Always
```

### ✅ `service.yaml` (Optional)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: csharp-console-app-service
spec:
  type: ClusterIP
  selector:
    app: csharp-console-app
  ports:
  - port: 80
    targetPort: 80
```

---

## 🔐 Step 3: GitHub Secrets Configuration

| Key                | Value                                                  |
|--------------------|--------------------------------------------------------|
| AZURE_CREDENTIALS  | JSON from `az ad sp create-for-rbac`                  |
| REGISTRY_USERNAME  | ACR or DockerHub username                             |
| REGISTRY_PASSWORD  | ACR or DockerHub password/token                       |
| REGISTRY_URL       | Your registry URL (e.g., `myregistry.azurecr.io`)     |
| AKS_RESOURCE_GROUP | Your AKS resource group name                          |
| AKS_CLUSTER_NAME   | Your AKS cluster name                                 |

Create a service principal:

```bash
az ad sp create-for-rbac --name "github-actions" --role contributor   --scopes /subscriptions/<sub_id>/resourceGroups/<resource_group>   --sdk-auth
```

---

## ⚙️ Step 4: GitHub Actions Workflow

Create `.github/workflows/deploy.yml`:

```yaml
name: Build and Deploy to AKS

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Set up Docker
      uses: docker/setup-buildx-action@v3

    - name: Log in to container registry
      uses: docker/login-action@v3
      with:
        registry: ${{ secrets.REGISTRY_URL }}
        username: ${{ secrets.REGISTRY_USERNAME }}
        password: ${{ secrets.REGISTRY_PASSWORD }}

    - name: Build and push Docker image
      run: |
        docker build -t ${{ secrets.REGISTRY_URL }}/csharp-console-app:latest .
        docker push ${{ secrets.REGISTRY_URL }}/csharp-console-app:latest

    - name: Set up Azure CLI
      uses: azure/login@v2
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: Set Kubernetes context
      uses: azure/aks-set-context@v3
      with:
        resource-group: ${{ secrets.AKS_RESOURCE_GROUP }}
        cluster-name: ${{ secrets.AKS_CLUSTER_NAME }}

    - name: Deploy to AKS
      run: |
        kubectl apply -f k8s/deployment.yaml
        kubectl apply -f k8s/service.yaml
```

---

## ✅ Step 5: Trigger Deployment

1. Create a PR from `dev` to `main`.
2. Merge it.
3. GitHub Actions will:
   - Build and push image
   - Connect to AKS
   - Deploy Kubernetes manifests

---

## 🧪 Step 6: Verification

Run these commands:

```bash
kubectl get deployments
kubectl get pods
kubectl logs <pod-name>
```

---

## ✅ Optional Improvements

- Use version tags instead of `latest`.
- Add Slack/webhook notifications.
- Use Helm for templated Kubernetes deployments.
