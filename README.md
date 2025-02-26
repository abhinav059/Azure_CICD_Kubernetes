# Azure_CICD_Kubernetes

# DevOps Project: Deploying MyHealth WebApp to Azure Kubernetes Service (AKS)

## Overview

This project demonstrates a CI/CD pipeline for deploying the **MyHealth WebApp**(https://github.com/piyushsachdeva/MyHealthClinic-AKS) to **Azure Kubernetes Service (AKS)** using **Azure DevOps**. The pipeline automates infrastructure provisioning, application deployment, and Kubernetes service exposure.

## **1. Pre-requisites**

a

- **Azure CLI**: `https://docs.microsoft.com/en-us/cli/azure/install-azure-cli`
- **kubectl**: `https://kubernetes.io/docs/tasks/tools/install-kubectl/`
- **Docker**: `https://docs.docker.com/get-docker/`
- **Azure DevOps Account**
- **Azure Subscription** with access to create resources

## **2. Infrastructure Provisioning**
![Image Description](https://github.com/yourusername/your-repo/blob/main/path-to-image.png](https://github.com/abhinav059/Images/blob/main/01.37.jpeg)


Use the following script to create the necessary infrastructure in **Azure**.

```bash
# Set variables
RESOURCE_GROUP="myhealth-rg"
AKS_CLUSTER="myhealth-aks"
ACR_NAME="myhealthacr"
SQL_SERVER="myhealth-sqlserver"
SQL_DB="myhealthdb"

# Create a Resource Group
az group create --name $RESOURCE_GROUP --location eastus

# Create Azure Container Registry (ACR)
az acr create --resource-group $RESOURCE_GROUP --name $ACR_NAME --sku Basic

# Create an AKS Cluster
az aks create --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER --node-count 2 --enable-addons monitoring --generate-ssh-keys

# Create SQL Server and Database
az sql server create --name $SQL_SERVER --resource-group $RESOURCE_GROUP --location eastus --admin-user adminuser --admin-password MySecurePassword123!
az sql db create --resource-group $RESOURCE_GROUP --server $SQL_SERVER --name $SQL_DB --service-objective S0
```
**![Image Description](https://github.com/abhinav059/Images/blob/main/09.jpeg)
**
## **3. Initializing the Azure DevOps Pipeline**

Add the following **Azure DevOps Pipeline YAML file** to your repository (`azure-pipelines.yml`).

```yaml
# Azure DevOps Pipeline for Deploying a Web App to Kubernetes (AKS)

trigger:
- main

pool:
  vmImage: ubuntu-latest

steps:
- task: replacetokens@4
  displayName: 'Replace tokens in appsettings.json'
  inputs:
    rootDirectory: '$(build.sourcesdirectory)/src/MyHealth.Web'
    targetFiles: 'appsettings.json'
    encoding: 'auto'
    tokenPattern: 'rm'
    writeBOM: true
    escapeType: 'none'
    actionOnMissing: 'warn'
    keepToken: false
    actionOnNoFiles: 'continue'
    enableTransforms: false
    enableRecursion: false
    useLegacyPattern: false
    enableTelemetry: true

- task: replacetokens@3
  displayName: 'Replace tokens in mhc-aks.yaml'
  inputs:
    targetFiles: 'mhc-aks.yaml'
    escapeType: none
    tokenPrefix: '__'
    tokenSuffix: '__'

- task: DockerCompose@0
  displayName: 'Run services'
  inputs:
    containerregistrytype: 'Azure Container Registry'
    azureSubscription: '$(AZURE_SUBSCRIPTION)'
    azureContainerRegistry: '{"loginServer":"$(ACR_LOGIN_SERVER)", "id" : "$(ACR_RESOURCE_ID)"}'
    dockerComposeFile: 'docker-compose.ci.build.yml'
    action: 'Run services'
    detached: false

- task: DockerCompose@0
  displayName: 'Build services'
  inputs:
    containerregistrytype: 'Azure Container Registry'
    azureSubscription: '$(AZURE_SUBSCRIPTION)'
    azureContainerRegistry: '{"loginServer":"$(ACR_LOGIN_SERVER)", "id" : "$(ACR_RESOURCE_ID)"}'
    dockerComposeFile: 'docker-compose.yml'
    dockerComposeFileArgs: 'DOCKER_BUILD_SOURCE='
    action: 'Build services'
    additionalImageTags: '$(Build.BuildId)'

- task: DockerCompose@0
  displayName: 'Push services'
  inputs:
    containerregistrytype: 'Azure Container Registry'
    azureSubscription: '$(AZURE_SUBSCRIPTION)'
    azureContainerRegistry: '{"loginServer":"$(ACR_LOGIN_SERVER)", "id" : "$(ACR_RESOURCE_ID)"}'
    dockerComposeFile: 'docker-compose.yml'
    dockerComposeFileArgs: 'DOCKER_BUILD_SOURCE='
    action: 'Push services'
    additionalImageTags: '$(Build.BuildId)'

- task: DockerCompose@0
  displayName: 'Lock services'
  inputs:
    containerregistrytype: 'Azure Container Registry'
    azureSubscription: '$(AZURE_SUBSCRIPTION)'
    azureContainerRegistry: '{"loginServer":"$(ACR_LOGIN_SERVER)", "id" : "$(ACR_RESOURCE_ID)"}'
    dockerComposeFile: 'docker-compose.yml'
    dockerComposeFileArgs: 'DOCKER_BUILD_SOURCE='
    action: 'Lock services'
    outputDockerComposeFile: '$(Build.StagingDirectory)/docker-compose.yml'

- task: CopyFiles@2
  displayName: 'Copy Files'
  inputs:
    Contents: |
      **/mhc-aks.yaml
      **/*.dacpac
    TargetFolder: '$(Build.ArtifactStagingDirectory)'

- task: PublishBuildArtifacts@1
  displayName: 'Publish Artifact'
  inputs:
    ArtifactName: deploy

```

## **4. Preparing Docker Files**

### **docker-compose.yml**

```yaml
version: '2'
services:
  myhealth.web:
    image: myhealth.web
    build:
      context: ./src/MyHealth.Web
      dockerfile: Dockerfile
```

### **docker-compose.override.yml**

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="15.0" Sdk="Microsoft.Docker.Sdk">
  <PropertyGroup Label="Globals">
    <ProjectVersion>2.0</ProjectVersion>
    <DockerTargetOS>Linux</DockerTargetOS>
    <DockerServiceUrl>http://localhost:{ServicePort}</DockerServiceUrl>
    <DockerServiceName>myhealth.web</DockerServiceName>
  </PropertyGroup>
  <ItemGroup>
    <None Include="docker-compose.yml" />
    <None Include="docker-compose.override.yml">
      <DependentUpon>docker-compose.yml</DependentUpon>
    </None>
  </ItemGroup>
</Project>
```

## **5. Pushing Docker Images to Azure Container Registry (ACR)**

```bash
az acr login --name $ACR_NAME

docker build -t $ACR_NAME.azurecr.io/myhealth.web:latest .
docker push $ACR_NAME.azurecr.io/myhealth.web:latest
```

## **6. Deploying to AKS**

Create a Kubernetes deployment file (`mhc-aks.yaml`) and apply it.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myhealth-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myhealth-web
  template:
    metadata:
      labels:
        app: myhealth-web
    spec:
      containers:
      - name: myhealth-web
        image: myhealthacr.azurecr.io/myhealth.web:latest
        ports:
        - containerPort: 80
```

Deploy to AKS:

```bash
kubectl apply -f mhc-aks.yaml
```

## **7. Verifying Deployment in AKS**

Connect to the AKS cluster and check running services.

```bash
az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER
kubectl get nodes
kubectl get pods
kubectl describe pod <pod-name>
```

## **8. Exposing the Application**

Expose the frontend service using a Load Balancer:

```bash
kubectl expose deployment myhealth-web --type=LoadBalancer --name=myhealth-service --port=80 --target-port=80
```

Check the external IP:

```bash
kubectl get service myhealth-service
```

## **9. Setting Up a Release Pipeline**

1. Go to **Azure DevOps → Pipelines → Releases**
2. Create a new release pipeline with an **Artifact Source** linked to your build pipeline.
3. Add a **Deploy to Kubernetes** task and select the AKS cluster.
4. Deploy the artifacts (`docker-compose.yml`, `mhc-aks.yaml`).

## **10. GitHub Repository Setup**

To publish this project on GitHub:

```bash
git init
git add .
git commit -m "Initial Commit"
git branch -M main
git remote add origin https://github.com/yourusername/myhealth-devops.git
git push -u origin main
```

## **Conclusion**

This DevOps pipeline automates the deployment of a web application to **Azure Kubernetes Service (AKS)** using **Azure DevOps**. It includes infrastructure provisioning, CI/CD setup, Docker containerization, and Kubernetes deployment. 🚀

