# 🚀 Azure Container Apps Deployment with Blue/Green, Rollback, and Shared Application Gateway

This guide explains how to build a **secure, scalable, and highly-available deployment system** for microservices on **Azure Container Apps (ACA)** using:

- **Azure DevOps Pipelines** for CI/CD  
- **Azure Container Registry (ACR)** for image storage  
- **Azure Application Gateway v2** (public + private frontends) for centralized ingress  
- **Blue/Green deployment strategy** with rollback  
- **Scheduled scaling** (e.g., auto-stop non-prod after hours)  

It mirrors an AWS setup like **ECR → ECS Fargate → ALB → CodeDeploy/CodePipeline**.

---

## 🧠 Architecture Overview

| Layer | Azure Component | Description |
|--------|------------------|--------------|
| Container Registry | **ACR** | Stores Docker images |
| Compute | **Azure Container Apps (ACA)** | Serverless container runtime (supports revisions + traffic split) |
| Ingress | **Application Gateway v2 (WAF)** | Shared L7 load balancer (host-based routing) |
| Networking | **VNET with subnets** | Segments AppGW and ACA |
| CI/CD | **Azure DevOps Pipelines** | Build, push, deploy, rollback |
| Scheduling | **KEDA Cron scaler** | Scales apps during office hours only |

---

## 📘 Design Overview

### Network Layout (Highly Available)
VNET (10.0.0.0/16)
├── snet-appgw (10.0.1.0/24) ← Application Gateway v2 (zones 1–3)
├── snet-aca (10.0.2.0/24) ← Container Apps Environment (internal, zones 1–3)
└── snet-shared(10.0.3.0/24) ← Optional shared resources

markdown
Copy code

- **App Gateway** is deployed with **zone redundancy (zones 1–3)**  
- **ACA Environment** is deployed as **internal**, also multi-zone  
- Each **Container App** decides if it’s:
  - `internal` → backend pool via **private frontend IP**
  - `public` → backend pool via **public frontend IP**

---

## 🏗️ 1. Networking & Core Infrastructure Setup

### 1.1 Create Resource Group
```bash
az group create -n rg-app-dev -l westeurope
1.2 Create Virtual Network and Subnets
bash
Copy code
az network vnet create -g rg-app-dev -n vnet-dev \
  --address-prefix 10.0.0.0/16 \
  --subnet-name snet-aca --subnet-prefix 10.0.2.0/24

az network vnet subnet create -g rg-app-dev --vnet-name vnet-dev \
  -n snet-appgw --address-prefix 10.0.1.0/24
Rule: App Gateway and ACA must have their own subnets. Azure requires dedicated subnets per managed service.

🧱 2. Create Azure Container Registry (ACR)
bash
Copy code
az acr create -n acrskumar01dev -g rg-app-dev --sku Premium --admin-enabled false
🧩 3. Create Azure Container Apps Environment (Internal)
bash
Copy code
az containerapp env create -n cae-dev -g rg-app-dev \
  --internal-only true \
  --zone-redundant true \
  --infrastructure-subnet-resource-id $(az network vnet subnet show \
      -g rg-app-dev --vnet-name vnet-dev -n snet-aca --query id -o tsv)
🐳 4. Create a Sample Container App (Internal Ingress)
bash
Copy code
az containerapp create -n app-orders-dev -g rg-app-dev \
  --environment cae-dev \
  --image nginx \
  --target-port 80 \
  --ingress internal \
  --min-replicas 1 --max-replicas 3
Retrieve internal FQDN:

bash
Copy code
az containerapp show -n app-orders-dev -g rg-app-dev --query properties.configuration.ingress.fqdn -o tsv
🌐 5. Create Application Gateway v2 (Public + Private Frontends)
5.1 Create Public and Private IPs
bash
Copy code
az network public-ip create -g rg-app-dev -n agw-pip-public --sku Standard
az network public-ip create -g rg-app-dev -n agw-pip-private --sku Standard --allocation-method Static --vnet-name vnet-dev
5.2 Create Application Gateway
bash
Copy code
az network application-gateway create -n agw-dev -g rg-app-dev \
  --sku WAF_v2 \
  --capacity 2 \
  --vnet-name vnet-dev \
  --subnet snet-appgw \
  --zones 1 2 3 \
  --public-ip-address agw-pip-public
App Gateway v2 automatically provides zone redundancy within the subnet.

🔀 6. Add Backend Pools (Container Apps)
Use internal FQDNs from ACA as backends (these are stable endpoints).

bash
Copy code
az network application-gateway address-pool create \
  -g rg-app-dev --gateway-name agw-dev \
  -n bp-orders-dev \
  --servers app-orders-dev.internal.<your-env>.azurecontainerapps.io
Repeat per app (e.g., bp-inventory-dev).

🔧 7. Add HTTP Settings and Health Probes
bash
Copy code
az network application-gateway http-settings create \
  -g rg-app-dev --gateway-name agw-dev \
  -n https-orders-dev \
  --port 443 --protocol Https \
  --host-name-from-backend-pool true \
  --pick-hostname-from-backend-address true \
  --timeout 30

az network application-gateway probe create \
  -g rg-app-dev --gateway-name agw-dev \
  -n probe-orders-dev \
  --protocol Https --path /healthz \
  --pick-hostname-from-backend-http-settings true
Attach probe:

bash
Copy code
az network application-gateway http-settings update \
  -g rg-app-dev --gateway-name agw-dev \
  -n https-orders-dev \
  --set probe.id=$(az network application-gateway probe show -g rg-app-dev -n probe-orders-dev --gateway-name agw-dev --query id -o tsv)
🌍 8. Create Listeners and Routing Rules
Public Listener (for public app)
bash
Copy code
az network application-gateway listener create \
  -g rg-app-dev --gateway-name agw-dev \
  -n orders-listener \
  --frontend-port 443 \
  --frontend-ip agw-pip-public \
  --host-names orders.dev.example.com \
  --ssl-cert orders-cert
Private Listener (for internal app)
bash
Copy code
az network application-gateway listener create \
  -g rg-app-dev --gateway-name agw-dev \
  -n inventory-listener \
  --frontend-port 443 \
  --frontend-ip agw-pip-private \
  --host-names inventory.dev.example.local \
  --ssl-cert internal-cert
Routing Rules
bash
Copy code
az network application-gateway rule create \
  -g rg-app-dev --gateway-name agw-dev \
  -n orders-rule \
  --http-listener orders-listener \
  --rule-type Basic \
  --address-pool bp-orders-dev \
  --http-settings https-orders-dev

az network application-gateway rule create \
  -g rg-app-dev --gateway-name agw-dev \
  -n inventory-rule \
  --http-listener inventory-listener \
  --rule-type Basic \
  --address-pool bp-inventory-dev \
  --http-settings https-inventory-dev
✅ Now your App Gateway has:

Public apps → public frontend listener

Internal apps → private frontend listener

🔄 9. Azure DevOps Pipeline (Blue/Green + 2hr Bake + Rollback)
Save this as azure-pipelines.yml:

yaml
Copy code
trigger:
  branches: [ main ]

parameters:
  - name: env
    type: string
    default: dev
    values: [ dev, preprod, prod ]

variables:
  serviceName: orders
  rgName: rg-app-$(env)
  acrName: acrskumar01$(env)
  appName: app-$(serviceName)-$(env)
  imageRepo: $(acrName).azurecr.io/$(serviceName)
  imageTag: $(Build.SourceVersion)

stages:
- stage: build_deploy
  displayName: Build, push, deploy (blue/green)
  jobs:
  - job: deploy
    pool: ubuntu-latest
    steps:
    - checkout: self

    - task: AzureContainerApps@1
      displayName: Build+Push+Deploy
      inputs:
        azureSubscription: 'sc-aca-$(parameters.env)'
        appSourcePath: '$(Build.SourcesDirectory)'
        acrName: '$(acrName)'
        containerAppName: '$(appName)'
        resourceGroup: '$(rgName)'

    - task: AzureCLI@2
      displayName: Capture old/new revisions
      inputs:
        azureSubscription: 'sc-aca-$(parameters.env)'
        scriptType: bash
        scriptLocation: inlineScript
        inlineScript: |
          ACTIVE_REVS=($(az containerapp revision list \
            --name $(appName) --resource-group $(rgName) \
            --query "[?properties.active].name" -o tsv))
          NEW_REV=${ACTIVE_REVS[-1]}
          OLD_REV=${ACTIVE_REVS[-2]}
          echo "##vso[task.setvariable variable=NEW_REV]$NEW_REV"
          echo "##vso[task.setvariable variable=OLD_REV]$OLD_REV"

    - task: AzureCLI@2
      displayName: Set traffic 100/0
      inputs:
        azureSubscription: 'sc-aca-$(parameters.env)'
        scriptType: bash
        scriptLocation: inlineScript
        inlineScript: |
          az containerapp ingress traffic set \
            --name $(appName) --resource-group $(rgName) \
            --revision-weight $(NEW_REV)=100 $(OLD_REV)=0

    - task: Delay@1
      displayName: Wait 120 minutes (bake)
      inputs:
        delayForMinutes: '120'

    - task: AzureCLI@2
      displayName: Deactivate old revision
      condition: and(succeeded(), ne(variables['OLD_REV'], ''))
      inputs:
        azureSubscription: 'sc-aca-$(parameters.env)'
        scriptType: bash
        scriptLocation: inlineScript
        inlineScript: |
          az containerapp revision deactivate \
            --name $(appName) --resource-group $(rgName) \
            --revision $(OLD_REV)

- stage: rollback
  displayName: Manual rollback
  condition: succeededOrFailed()
  jobs:
  - job: rollback
    pool: ubuntu-latest
    steps:
    - task: AzureCLI@2
      displayName: Roll traffic back to OLD
      inputs:
        azureSubscription: 'sc-aca-$(parameters.env)'
        scriptType: bash
        scriptLocation: inlineScript
        inlineScript: |
          ACTIVE_REVS=($(az containerapp revision list \
            --name $(appName) --resource-group $(rgName) \
            --query "[?properties.active].name" -o tsv))
          NEW_REV=${ACTIVE_REVS[-1]}
          OLD_REV=${ACTIVE_REVS[-2]}
          az containerapp ingress traffic set \
            --name $(appName) --resource-group $(rgName) \
            --revision-weight $OLD_REV=100 $NEW_REV=0
⏰ 10. Scheduled Scaling (KEDA Cron)
To stop non-prod environments after 5 PM:

yaml
Copy code
properties:
  template:
    scale:
      minReplicas: 0
      maxReplicas: 5
      rules:
        - name: office-hours-cron
          custom:
            type: cron
            metadata:
              timezone: Europe/London
              start: "0 9 * * 1-5"     # Scale up at 9AM
              end:   "0 17 * * 1-5"    # Scale down at 5PM
              desiredReplicas: "1"
Apply via:

bash
Copy code
az containerapp update -n app-orders-dev -g rg-app-dev --yaml scale.yaml
🔐 11. Security Best Practices
Disable ACR Admin User

Use Managed Identities for image pulls (AcrPull role)

Use Workload Identity Federation for DevOps pipelines (no secrets)

Restrict App Gateway inbound with NSGs/WAF rules

Use Private DNS Zone for internal ACA FQDNs

🧱 12. Terraform Next Steps (Optional)
Once verified, export configuration:

bash
Copy code
az containerapp show -n app-orders-dev -g rg-app-dev > app.json
az network application-gateway show -n agw-dev -g rg-app-dev > appgw.json
Model resources using:

azurerm_container_app_environment

azurerm_container_app

azurerm_application_gateway

azurerm_container_registry

Parameterize:

env, location, appName, acrName, scale_rules

✅ Final Architecture
pgsql
Copy code
                  ┌──────────────────────┐
                  │     Internet (443)   │
                  └─────────┬────────────┘
                            │
               ┌───────────────────────────┐
               │ Application Gateway v2     │
               │ Public & Private Frontends │
               │ snet-appgw, zones 1–3      │
               └─────────┬──────────────────┘
           ┌─────────────┴──────────────┐
           │                            │
  Public Traffic                  Internal Traffic
(orders.example.com)             (inventory.example.local)
           │                            │
 ┌──────────────────┐         ┌──────────────────┐
 │ Container App:   │         │ Container App:   │
 │ orders-dev       │         │ inventory-dev    │
 │ internal ingress │         │ internal ingress │
 └──────────────────┘         └──────────────────┘
           │                            │
      Container Apps Environment (Internal, Multi-Zone)
                       │
                    [ACR + Log Analytics + VNET]
🧠 Key Takeaways
1 subnet per service, not per zone — HA is achieved via zone redundancy

App Gateway v2 + ACA internal ingress = shared L7 load balancer model

Revisions + traffic weights enable blue/green & instant rollback

KEDA cron controls cost for non-prod

DevOps pipeline automates the entire CI/CD loop with no secrets
