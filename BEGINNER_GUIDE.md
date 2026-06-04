# Beginner's Complete Guide — DevOps Real-World Project on AWS

This guide walks you through the entire course from scratch, explaining **what every step does and why**, written for someone who is brand new to DevOps.

## Guide Contents

| Module | Topic | What you'll understand |
|--------|-------|----------------------|
| 02 | Docker on EC2 | Every install command explained line by line |
| 03 | Dockerfiles | What `FROM`, `RUN`, `CMD`, `EXPOSE` etc. each do |
| 04 | Docker Compose | Multi-container apps, volumes, health checks |
| 05 | Buildx | Multi-platform images for AMD64 + ARM64 |
| 06 | Terraform Basics | Providers, variables, resources, remote state |
| 07 | EKS Cluster | VPC → EKS flow, kubeconfig setup, destroy order |
| 08 | Kubernetes | Pods, Deployments, Services, ConfigMaps all explained |
| 09 | Secrets | AWS Secrets Manager + CSI driver |
| 10 | Storage | EBS CSI, PersistentVolumeClaims |
| 11 | Ingress | ALB controller, path-based routing |
| 12 | Helm | Package manager, chart structure, values overrides |
| 13 | EKS + Add-Ons | All add-ons auto-installed via Terraform |
| 14 | Full Retail Store | All 5 microservices wired to AWS data plane |
| 15-16 | ExternalDNS | Friendly DNS names for your app |
| 17 | Karpenter | Node autoscaling |
| 18 | HPA | Pod replica autoscaling |
| 19 | Helm full app | One `helm install` deploys everything |
| 20 | Observability | Metrics, Logs, Traces with ADOT |
| 21 | CI/CD | GitHub Actions + ArgoCD GitOps pipeline |

---

## What Is This Project?

This project teaches you how a real company would deploy a **Retail Store application** (think: an e-commerce site with a catalog, cart, checkout, and orders) to the cloud using modern DevOps tools. By the end you will know how to:

- Package apps with **Docker**
- Provision cloud infrastructure with **Terraform**
- Orchestrate containers with **Kubernetes (AWS EKS)**
- Package Kubernetes apps with **Helm**
- Automate deployments with **CI/CD (GitHub Actions + ArgoCD)**

---

## Prerequisites — Install These Before You Start

| Tool | Why You Need It | Install Link |
|------|----------------|-------------|
| AWS CLI | Talk to AWS from your terminal | https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html |
| Terraform | Create cloud infrastructure with code | https://developer.hashicorp.com/terraform/install |
| kubectl | Control Kubernetes clusters | https://kubernetes.io/docs/tasks/tools/ |
| Helm | Install Kubernetes apps like packages | https://helm.sh/docs/intro/install/ |
| Docker Desktop (optional) | Run containers locally | https://www.docker.com/products/docker-desktop |

Verify everything works:

```bash
aws --version        # should print aws-cli/2.x.x
terraform --version  # should print Terraform v1.x.x
kubectl version --client
helm version
docker --version
```

---

## Module 02 — Docker on AWS EC2

### What is Docker?
Docker lets you **package an application and all its dependencies into a single portable box** called a container. You can run this box anywhere — your laptop, a server, or the cloud — and it always behaves the same way.

### Step 1: Launch an EC2 Instance

In the AWS Console:
1. Go to **EC2 → Launch Instance**
2. Choose **Amazon Linux 2023** as the operating system
3. Pick instance type **t3.large** (2 vCPU, 8 GB RAM — big enough for Docker)
4. Add 30 GB storage
5. Create or select an SSH key pair (you'll need the `.pem` file to connect)
6. In Security Group, open port **22** (SSH), **80** (HTTP), **8080** (app)

### Step 2: Connect to Your EC2 Instance

```bash
# Replace your-key.pem and your EC2 IP address
ssh -i your-key.pem ec2-user@<your-ec2-public-ip>
```

`ssh` opens a secure terminal session to your remote server. The `-i` flag specifies your private key file.

### Step 3: Install Docker

```bash
sudo dnf update -y              # update all OS packages first
sudo dnf install docker -y      # install docker from the package manager
sudo systemctl enable docker    # make docker start automatically when the server reboots
sudo systemctl start docker     # start docker right now
sudo usermod -aG docker ec2-user  # add yourself to the docker group so you don't need sudo
```

**Log out and log back in** after the last command so the group change takes effect.

### Step 4: Test Docker

```bash
docker version          # shows Client and Server versions — both must appear
docker run hello-world  # downloads a tiny test image and runs it
docker images           # lists all images downloaded to this machine
```

`docker run hello-world` does three things automatically:
1. Checks if the `hello-world` image exists locally — it doesn't
2. Downloads (pulls) it from Docker Hub
3. Runs it and prints a welcome message, then stops

---

## Module 03 — Dockerfiles

### What is a Dockerfile?
A **Dockerfile** is a text file that is a recipe for building a Docker image. Each line is an instruction.

```dockerfile
FROM node:18-alpine
# FROM — start from an existing base image (Node.js 18 on a tiny Alpine Linux OS)
# This means you don't need to install Node.js yourself

WORKDIR /app
# WORKDIR — set the working directory inside the container
# All following commands run from /app

COPY package*.json ./
# COPY — copy files from your machine into the container
# We copy package.json first (before the source code) for caching reasons

RUN npm install
# RUN — execute a command while building the image
# This installs all Node.js dependencies

COPY . .
# Copy the rest of the source code into the container

EXPOSE 8080
# EXPOSE — document which port the app listens on (does not actually open it)

CMD ["node", "server.js"]
# CMD — the default command to run when a container starts
# Only one CMD allowed — it starts your application
```

Build and run it:

```bash
# Build an image from the Dockerfile in the current folder, tag it "myapp:1.0"
docker build -t myapp:1.0 .

# Run the image as a container, mapping your port 8080 to the container's port 8080
docker run -p 8080:8080 myapp:1.0
```

---

## Module 04 — Docker Compose

### What is Docker Compose?
When your app has **multiple containers that need to work together** (e.g., a web server + a database), Docker Compose lets you define and start all of them with a single command.

```yaml
# docker-compose.yaml
version: "3.8"

services:

  catalog:                          # name of this service
    image: retail-store-catalog:1.0 # which Docker image to use
    ports:
      - "8080:8080"                 # host:container port mapping
    depends_on:
      db:
        condition: service_healthy  # wait for DB to be healthy before starting catalog
    environment:
      DB_HOST: db                   # pass environment variables into the container

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: catalog
    volumes:
      - db-data:/var/lib/mysql      # persist database files to a named volume
    healthcheck:
      test: ["CMD", "mysqladmin", "ping"]
      interval: 10s
      retries: 5

volumes:
  db-data:                          # declare the named volume
```

```bash
docker compose up -d      # start all services in the background (-d = detached)
docker compose ps         # check status of all services
docker compose logs -f    # stream logs from all services
docker compose down       # stop and remove all containers
```

---

## Module 05 — Docker Buildx (Multi-Platform Images)

### What is Buildx?
By default, Docker builds images for your current CPU architecture (e.g., AMD64 on most laptops). Buildx lets you build images that work on **multiple CPU architectures** (AMD64 and ARM64) in one command — important because AWS Graviton nodes use ARM64.

```bash
# Create a new builder that supports multi-platform builds
docker buildx create --name mybuilder --use

# Build for both AMD64 (Intel/AMD) and ARM64 (Apple M1, AWS Graviton)
# --push sends the result directly to Docker Hub
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t yourdockerhubuser/myapp:1.0 \
  --push \
  .
```

---

## Module 06 — Terraform Basics

### What is Terraform?
Terraform lets you write code that **creates and manages cloud infrastructure**. Instead of clicking around the AWS Console, you write `.tf` files and Terraform does the clicking for you — repeatably and consistently.

### Core Terraform Commands

```bash
terraform init      # download provider plugins (like the AWS plugin)
terraform validate  # check your .tf files for syntax errors
terraform plan      # preview what Terraform WILL create/change/destroy — no changes yet
terraform apply     # actually create the infrastructure (asks for confirmation)
terraform destroy   # delete everything Terraform created
```

### Key Terraform Concepts

**Providers** — tell Terraform which cloud to talk to:

```hcl
# c1_versions.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"   # use any AWS provider version 5.x
    }
  }
}

provider "aws" {
  region = "us-east-1"   # which AWS region to create resources in
}
```

**Variables** — make your code reusable:

```hcl
# c2_variables.tf
variable "aws_region" {
  description = "AWS region to deploy resources"
  type        = string
  default     = "us-east-1"
}

variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"  # this gives you 65,536 IP addresses
}
```

**Resources** — the actual infrastructure to create:

```hcl
# c3_vpc.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr          # use the variable defined above
  enable_dns_hostnames = true                  # allow EC2 instances to have DNS names
  enable_dns_support   = true

  tags = {
    Name = "production-vpc"
  }
}
```

**Remote Backend** — store Terraform state in S3 so teams can share it:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"  # S3 bucket to store the state file
    key            = "vpc/terraform.tfstate"       # path inside the bucket
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"        # prevent two people running apply at once
  }
}
```

**VPC (Virtual Private Cloud)** — your private network in AWS with public and private subnets:

```
Internet → Internet Gateway → Public Subnet (Load Balancers) → Private Subnet (EC2/EKS nodes)
```

---

## Module 07 — EKS Cluster with Terraform

### What is EKS?
**Amazon EKS (Elastic Kubernetes Service)** is AWS's managed Kubernetes service. AWS runs the Kubernetes control plane (the brain of the cluster) for you. You just define worker nodes.

### Project Structure

```
07_Terraform_EKS_Cluster/
├── 01_VPC_terraform-manifests/    ← first create the VPC
└── 02_EKS_terraform-manifests/    ← then create the EKS cluster inside that VPC
```

The EKS project reads the VPC outputs using a **remote state datasource**:

```hcl
# c3_remote-state.tf
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state-bucket"
    key    = "vpc/terraform.tfstate"  # read the VPC state file
    region = "us-east-1"
  }
}
# Now you can use data.terraform_remote_state.vpc.outputs.vpc_id anywhere
```

### Steps to Create the Cluster

```bash
# Step 1: Create the VPC first
cd 01_VPC_terraform-manifests
terraform init
terraform apply -auto-approve  # -auto-approve skips the yes/no confirmation

# Step 2: Create the EKS cluster (uses the VPC created above)
cd ../02_EKS_terraform-manifests
terraform init
terraform apply -auto-approve

# Step 3: Configure kubectl to connect to your new cluster
aws eks update-kubeconfig --name <cluster_name> --region us-east-1
# This writes the cluster credentials to ~/.kube/config

# Step 4: Verify the cluster
kubectl get nodes    # should show your worker nodes as "Ready"
kubectl get pods -n kube-system  # should show system pods running
```

### Clean Up (to avoid AWS charges)

```bash
# IMPORTANT: Always destroy EKS before destroying VPC (reverse order)
cd 02_EKS_terraform-manifests && terraform destroy -auto-approve
cd ../01_VPC_terraform-manifests && terraform destroy -auto-approve
```

---

## Module 08 — Kubernetes Fundamentals

### What is Kubernetes?
Kubernetes (K8s) is a system that **automatically manages containers** — it decides which server to run them on, restarts them if they crash, scales them up when traffic increases, and routes network traffic to them.

### Core Concepts

#### Pods (08_01)
A **Pod** is the smallest deployable unit in Kubernetes — it wraps one or more containers.

```yaml
# 01_catalog_pod.yaml
apiVersion: v1           # which Kubernetes API version to use
kind: Pod                # what type of resource this is
metadata:
  name: catalog-pod      # name of this pod
  labels:
    app: catalog         # labels are key-value tags used for selection
spec:
  containers:
    - name: catalog
      image: "public.ecr.aws/aws-containers/retail-store-sample-catalog:1.3.0"
      ports:
        - containerPort: 8080   # the port the app listens on inside the container
      resources:
        requests:               # minimum resources Kubernetes will guarantee
          cpu: "100m"           # 100 millicores = 0.1 CPU cores
          memory: "128Mi"       # 128 megabytes
        limits:                 # maximum resources the container can use
          cpu: "250m"
          memory: "256Mi"
      readinessProbe:           # Kubernetes checks this before sending traffic
        httpGet:
          path: /health         # call GET /health
          port: 8080            # on port 8080
        # if the response is 200 OK, the pod is "ready"
```

```bash
kubectl apply -f 01_catalog_pod.yaml   # create the pod
kubectl get pods                        # see its status
kubectl describe pod catalog-pod        # detailed info + events (great for debugging)
kubectl logs catalog-pod                # see the app's log output
kubectl port-forward pod/catalog-pod 8080:8080  # access it from your laptop
kubectl exec -it catalog-pod -- /bin/sh  # open a shell inside the container
kubectl delete pod catalog-pod           # delete the pod
```

#### Deployments (08_02)
A **Deployment** manages multiple copies (replicas) of a Pod and handles rolling updates.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog-deployment
spec:
  replicas: 2            # run 2 identical pods at all times
  selector:
    matchLabels:
      app: catalog       # this deployment manages pods with label app=catalog
  template:              # pod template — same as a pod spec
    metadata:
      labels:
        app: catalog
    spec:
      containers:
        - name: catalog
          image: "public.ecr.aws/aws-containers/retail-store-sample-catalog:1.3.0"
```

```bash
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl get pods           # should now show 2 pods
kubectl rollout status deployment/catalog-deployment
kubectl set image deployment/catalog-deployment catalog=catalog:2.0  # rolling update
kubectl rollout undo deployment/catalog-deployment                   # rollback
```

#### Services (08_03)
A **Service** gives your pods a stable DNS name and IP address, and load-balances traffic across all pod replicas.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: catalog-service
spec:
  type: ClusterIP          # ClusterIP = only reachable inside the cluster
  selector:
    app: catalog           # send traffic to pods with this label
  ports:
    - port: 80             # port the service listens on
      targetPort: 8080     # port the pods actually listen on
```

Service types:
- **ClusterIP** — internal only (default)
- **NodePort** — accessible on each node's IP + a port
- **LoadBalancer** — creates an AWS load balancer, publicly accessible

#### ConfigMaps (08_04)
**ConfigMaps** store configuration data (environment variables, config files) separately from your container image.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: catalog-config
data:
  DB_HOST: "catalog-db.example.com"    # key: value pairs
  DB_PORT: "3306"
  LOG_LEVEL: "info"
```

Reference it in a pod:

```yaml
envFrom:
  - configMapRef:
      name: catalog-config   # inject all keys as environment variables
```

---

## Module 09 — Kubernetes Secrets

### What are Kubernetes Secrets?
Secrets store sensitive data (passwords, API keys) separately from your app code. With AWS Secrets Manager + the CSI driver, secrets are pulled directly from AWS rather than stored in Kubernetes (which is more secure).

```bash
# Install the EKS Pod Identity Agent add-on first
# This lets pods assume AWS IAM roles securely

# Then install the AWS Secrets Manager CSI Driver
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --namespace kube-system

# Install the AWS provider for the CSI driver
kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml
```

A **SecretProviderClass** tells the CSI driver which AWS secret to fetch:

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: catalog-db-secret
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "catalog/db/password"   # name of the secret in AWS Secrets Manager
        objectType: "secretsmanager"
```

---

## Module 10 — Kubernetes Storage

### EBS CSI Driver
The **EBS CSI (Container Storage Interface) Driver** lets Kubernetes create and manage AWS EBS (Elastic Block Store) volumes automatically.

```bash
# Install via EKS add-on
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name aws-ebs-csi-driver
```

**PersistentVolumeClaim** — request storage for a pod:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce          # one node can read/write at a time
  storageClassName: gp3      # use AWS gp3 EBS volumes
  resources:
    requests:
      storage: 10Gi          # request 10 gigabytes
```

---

## Module 11 — Kubernetes Ingress

### What is Ingress?
An **Ingress** is a smart HTTP router that sits in front of your services. One AWS Load Balancer can route to multiple services based on URL paths or hostnames.

```
Internet → AWS ALB (Load Balancer) → Ingress → /catalog → catalog-service
                                              → /cart    → cart-service
                                              → /        → ui-service
```

### Install AWS Load Balancer Controller

```bash
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### Ingress Manifest

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: retail-store-ingress
  annotations:
    kubernetes.io/ingress.class: alb           # use the AWS ALB controller
    alb.ingress.kubernetes.io/scheme: internet-facing  # create a public load balancer
    alb.ingress.kubernetes.io/target-type: ip   # route directly to pod IPs
spec:
  rules:
    - http:
        paths:
          - path: /catalog
            pathType: Prefix
            backend:
              service:
                name: catalog-service
                port:
                  number: 80
```

---

## Module 12 — Helm

### What is Helm?
**Helm** is the package manager for Kubernetes — like `npm` for Node.js or `pip` for Python. It lets you install, upgrade, and uninstall complex Kubernetes apps with a single command.

### Key Commands

```bash
helm repo add <name> <url>     # add a chart repository (like adding an npm registry)
helm repo update               # fetch the latest list of charts
helm search repo <keyword>     # search for charts
helm install <release-name> <chart>  # install a chart (creates all K8s resources)
helm list                      # see installed releases
helm upgrade <release-name> <chart>  # upgrade to a new version
helm uninstall <release-name>  # delete the release and all its resources
helm template <chart>          # render the templates without installing (great for debugging)
```

### Chart Structure

```
my-chart/
├── Chart.yaml          ← metadata: name, version, description
├── values.yaml         ← default configuration values
└── templates/          ← Kubernetes YAML templates with {{ .Values.xxx }} placeholders
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

`values.yaml` example:

```yaml
replicaCount: 2

image:
  repository: public.ecr.aws/aws-containers/retail-store-sample-catalog
  tag: "1.3.0"

service:
  type: ClusterIP
  port: 80
```

Override values at install time:

```bash
# Override a single value on the command line
helm install catalog ./my-chart --set replicaCount=3

# Or supply a custom values file
helm install catalog ./my-chart -f my-values.yaml
```

---

## Module 13 — EKS Cluster with Add-Ons

This module creates the same VPC + EKS cluster as Module 07, but uses **Terraform EKS Blueprints** to install common add-ons automatically:

| Add-On | Purpose |
|--------|---------|
| VPC CNI | Pod networking inside the VPC |
| CoreDNS | DNS resolution between pods |
| kube-proxy | Network routing rules |
| EBS CSI Driver | Persistent storage |
| AWS Load Balancer Controller | Ingress/ALB management |
| Pod Identity Agent | Secure IAM role assumption for pods |

```bash
# create-cluster.sh — spins up VPC + EKS + all add-ons
./create-cluster.sh

# destroy-cluster.sh — tears everything down in the correct order
./destroy-cluster.sh
```

---

## Module 14 — Full Retail Store App on AWS Data Plane

This is where everything comes together. The retail store application has **5 microservices**:

| Service | Language | Data Store |
|---------|----------|-----------|
| UI | Node.js | — |
| Catalog | Go | MySQL (RDS) |
| Cart | Java | DynamoDB |
| Checkout | Node.js | Redis (ElastiCache) |
| Orders | Java | PostgreSQL (RDS) |

### Infrastructure created by Terraform (14_01)

```bash
cd 14_RetailStore_Microservices_with_AWS_Data_Plane/14_01_RetailStore_AWS_Data_Plane
terraform init
terraform apply -auto-approve
# Creates: RDS MySQL, RDS PostgreSQL, DynamoDB table, ElastiCache Redis, IAM roles, Secrets Manager secrets
```

### Kubernetes deployment (14_02)

Each microservice has its own folder with these files:

```
01_catalog/
├── 01_catalog_service_account.yaml   ← identity for the pod to assume an AWS IAM role
├── 02_catalog_configmap.yaml         ← non-secret config (DB host, ports)
├── 03_catalog_deployment.yaml        ← the actual application pod
└── 04_catalog_clusterip_service.yaml ← internal DNS name for other pods to reach it
```

Apply all manifests in order:

```bash
# Apply secrets provider first, then services
kubectl apply -f 01_secretproviderclass/
kubectl apply -f 02_RetailStore_Microservices/01_catalog/
kubectl apply -f 02_RetailStore_Microservices/02_cart/
kubectl apply -f 02_RetailStore_Microservices/03_checkout/
kubectl apply -f 02_RetailStore_Microservices/04_orders/
kubectl apply -f 02_RetailStore_Microservices/05_ui/
kubectl apply -f 03_Ingress/

# Verify all pods are running
kubectl get pods -n retailstore
kubectl get ingress -n retailstore  # get the public URL
```

---

## Module 15 & 16 — ExternalDNS

**ExternalDNS** automatically creates Route 53 DNS records when you create a Kubernetes Ingress or Service. Instead of copying the ugly auto-generated ALB URL, you get a friendly name like `shop.example.com`.

```bash
helm install external-dns bitnami/external-dns \
  --set provider=aws \
  --set aws.region=us-east-1 \
  --set domainFilters[0]=example.com    # only manage records for this domain
```

---

## Module 17 — Autoscaling with Karpenter

**Karpenter** is a node autoscaler — when a pod can't be scheduled because there are no nodes with enough resources, Karpenter automatically **launches new EC2 instances** in seconds.

```yaml
# NodePool — tells Karpenter what kind of nodes to create
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64", "arm64"]   # allow both CPU architectures
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand", "spot"] # prefer spot instances (cheaper)
  limits:
    cpu: 100                           # don't create more than 100 total CPUs
  disruption:
    consolidationPolicy: WhenUnderutilized  # remove nodes when pods move off them
```

---

## Module 18 — Horizontal Pod Autoscaler (HPA)

**HPA** scales the **number of pod replicas** based on CPU or memory usage — different from Karpenter which scales nodes.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: catalog-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: catalog-deployment
  minReplicas: 2       # always keep at least 2 pods
  maxReplicas: 10      # never exceed 10 pods
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50   # scale up when average CPU > 50%
```

---

## Module 19 — Helm for the Full Retail Store

Instead of applying individual YAML files (Module 14 approach), this module uses **Helm charts** to deploy all microservices:

```bash
# Install the entire retail store app with one command
helm install retailstore . \
  --namespace retailstore \
  --create-namespace \
  -f values-aws-dataplane.yaml

# Upgrade to a new version
helm upgrade retailstore . -f values-aws-dataplane.yaml

# Uninstall everything
helm uninstall retailstore -n retailstore
```

---

## Module 20 — Observability with OpenTelemetry

**Observability** means being able to understand what your system is doing from the outside by looking at three types of data:

| Signal | What It Tells You | Tool |
|--------|------------------|------|
| **Metrics** | How fast, how many errors, CPU usage | AWS CloudWatch |
| **Logs** | What happened and when | CloudWatch Logs |
| **Traces** | How a single request traveled through all microservices | AWS X-Ray |

**ADOT (AWS Distro for OpenTelemetry)** is an AWS-managed collector that gathers all three signals from your pods and sends them to AWS services.

```bash
# Install ADOT via EKS add-on
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name adot
```

---

## Module 21 — CI/CD with GitHub Actions + ArgoCD

### The Full GitOps Pipeline

```
Developer pushes code
       ↓
GitHub Actions (CI)
  - builds Docker image
  - runs tests
  - pushes image to ECR
  - updates Helm chart version in Git
       ↓
ArgoCD (CD) detects Git change
  - pulls new Helm chart
  - deploys to EKS cluster
       ↓
App updated with zero downtime
```

### GitHub Actions Workflow (`.github/workflows/ci.yaml`)

```yaml
name: CI Pipeline

on:
  push:
    branches: [main]    # trigger when code is pushed to main branch

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4   # check out the code

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}  # use OIDC, not access keys
          aws-region: us-east-1

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push Docker image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}       # use the git commit hash as the image tag
        run: |
          docker build -t $ECR_REGISTRY/catalog:$IMAGE_TAG .
          docker push $ECR_REGISTRY/catalog:$IMAGE_TAG
```

### ArgoCD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: retail-store
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/your-gitops-repo
    targetRevision: HEAD
    path: helm/retail-store           # folder containing the Helm chart
  destination:
    server: https://kubernetes.default.svc
    namespace: retailstore
  syncPolicy:
    automated:
      prune: true      # delete K8s resources that are removed from Git
      selfHeal: true   # revert any manual changes in the cluster
```

---

## Cost Warning — Always Clean Up!

EKS, RDS, ElastiCache, and NAT Gateways cost money **even when you're not using them**. After each module, run destroy:

```bash
# For Terraform modules:
terraform destroy -auto-approve

# For Kubernetes resources:
kubectl delete namespace retailstore

# For Helm releases:
helm uninstall retailstore -n retailstore
```

Use the provided scripts when available:

```bash
./destroy-cluster.sh   # destroys EKS cluster and VPC in the correct order
```

---

## Quick Reference — Most Used Commands

```bash
# --- Terraform ---
terraform init && terraform plan && terraform apply -auto-approve
terraform destroy -auto-approve

# --- kubectl ---
kubectl get pods -A                     # list all pods in all namespaces
kubectl get pods -n <namespace>         # list pods in a specific namespace
kubectl describe pod <pod-name>         # detailed pod info + events
kubectl logs <pod-name> -f              # stream logs
kubectl exec -it <pod-name> -- /bin/sh  # shell into a container
kubectl apply -f <file-or-folder>/      # create/update resources
kubectl delete -f <file-or-folder>/     # delete resources

# --- Helm ---
helm install <name> <chart>
helm upgrade <name> <chart>
helm uninstall <name>
helm list -A

# --- AWS CLI ---
aws eks update-kubeconfig --name <cluster> --region us-east-1
aws eks list-clusters
```

---

*Follow the module order (02 → 21) for the best learning experience. Each module builds on the previous one.*
