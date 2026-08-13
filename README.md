# GitOps EKS Production Architecture

Welcome to the production GitOps repository! This repository contains both the Infrastructure as Code (Terraform) to build a secure AWS EKS cluster, and the GitOps manifests (Helm/Argo CD) to deploy the microservices.

## 🌟 High-Level Architecture (Demo View)
This diagram illustrates the high-level GitOps workflow and traffic routing, perfect for understanding the value of the platform.

```mermaid
flowchart LR
    %% Colors and Styles
    classDef user fill:#4CAF50,stroke:#388E3C,stroke-width:2px,color:white;
    classDef git fill:#24292E,stroke:#000,stroke-width:2px,color:white;
    classDef argocd fill:#EF7B45,stroke:#D84315,stroke-width:2px,color:white;
    classDef aws fill:#232F3E,stroke:#FF9900,stroke-width:2px,color:white;
    classDef db fill:#336791,stroke:#2c3e50,stroke-width:2px,color:white;
    classDef vault fill:#000000,stroke:#607D8B,stroke-width:2px,color:white;

    Dev(("👨‍💻 Developer")):::user
    User(("👤 End User")):::user
    
    Git["🐙 GitHub\n(Source of Truth)"]:::git
    Argo["🦑 Argo CD\n(GitOps Engine)"]:::argocd
    ALB["🌐 AWS Load Balancer"]:::aws
    
    subgraph EKS ["Kubernetes (AWS EKS)"]
        UI["🎨 Guestbook UI"]:::aws
        API["⚙️ Backend API"]:::aws
        DB[("🐘 Postgres DB")]:::db
        Vault["🗄️ HashiCorp Vault\n(Secrets)"]:::vault
    end

    %% GitOps Flow
    Dev == "1. Pushes Code/Config" ===> Git
    Git -. "2. Monitors Changes" .-> Argo
    Argo == "3. Automatically Deploys" ===> UI
    Argo == "3. Automatically Deploys" ===> API
    Argo == "3. Automatically Deploys" ===> Vault

    %% Traffic Flow
    User == "Visits Website" ===> ALB
    ALB -->|/| UI
    ALB -->|/api| API
    API -->|Reads Data| DB
    
    %% Secrets Flow
    Vault -. "4. Injects Passwords\n(External Secrets)" .-> DB
```

## 🛠️ Network Architecture (Engineering View)
This diagram illustrates the deep-dive AWS network topology, VPC boundaries, and exact component placement.

```mermaid
flowchart TD
    subgraph AWS["☁️ AWS Cloud (eu-north-1)"]
        subgraph VPC["VPC (10.0.0.0/16)"]
            
            subgraph Public["Public Subnets"]
                ALB["🌐 AWS Application Load Balancer"]
                NAT["🔀 NAT Gateway"]
            end

            subgraph Private["Private Subnets (EKS Worker Nodes)"]
                subgraph EKS["EKS Cluster: gitops-prod-cluster"]
                    ArgoCD["🐙 Argo CD (Controller)"]
                    ALBController["⚙️ AWS ALB Controller"]
                    ESO["🔐 External Secrets Operator"]
                    Vault["🗄️ HashiCorp Vault"]
                    
                    subgraph Apps["Workloads"]
                        Guestbook["🎨 Guestbook UI (/)"]
                        BackendAPI["⚙️ Backend API (/api)"]
                        Postgres["🐘 Postgres DB"]
                    end
                end
            end
        end
    end

    User(("👤 User")) -->|HTTPS Traffic| ALB
    ALB -->|Routes '/'| Guestbook
    ALB -->|Routes '/api'| BackendAPI
    BackendAPI -->|Internal TCP| Postgres
    
    GitHub[("🐈‍⬛ GitHub Repository")]
    
    ArgoCD -.->|1. Pulls Manifests| GitHub
    ArgoCD -.->|2. Synchronizes| Apps
    ArgoCD -.->|2. Synchronizes| Vault
    
    ALBController -.->|Listens for Ingress & Creates| ALB
    ESO -.->|Fetches DB Password| Vault
    ESO -.->|Injects Native k8s Secret| Postgres
```

---

## 1. AWS Authentication
Before running any Terraform, you must authenticate your terminal with AWS.
```bash
# If using AWS IAM Identity Center (SSO):
aws sso login

# Or verify existing credentials:
aws sts get-caller-identity
```

## 2. Infrastructure as Code (Terraform)
This repository uses Terraform to provision the AWS VPC and a fully private EKS cluster. The state is securely stored in an AWS S3 backend (`acme-corp-terraform-state-e31e6482`).

```bash
cd terraform/
terraform init
terraform plan
terraform apply
```
*Note: Provisioning the EKS cluster and Node Groups takes approximately 15-20 minutes.*

## 3. Connect to the Cluster
Once Terraform completes, update your local `kubeconfig` to talk to the new cluster:
```bash
aws eks update-kubeconfig --region us-east-1 --name gitops-prod-cluster
```

## 4. Install Argo CD (The GitOps Engine)
Argo CD is the only tool we install manually. Once installed, it takes over and deploys everything else in this repository automatically.
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

## 5. Deploy Workloads
Apply the root ApplicationSet. Argo CD will instantly read the `infrastructure/` and `workloads/` directories in this repo and spin up all microservices, Vault, and the External Secrets Operator.
```bash
kubectl apply -f infrastructure/argocd/workloads-appset.yaml
```

## 6. Secrets Management
We follow the strict GitOps rule: **Never commit passwords to Git.**
- **HashiCorp Vault** is deployed automatically by Argo CD.
- **External Secrets Operator (ESO)** is deployed to read from Vault.
- The `postgres` and `backend-api` Helm charts contain `ExternalSecret` templates that instruct ESO to fetch the `db-password` from Vault at runtime and inject it into the pods.
