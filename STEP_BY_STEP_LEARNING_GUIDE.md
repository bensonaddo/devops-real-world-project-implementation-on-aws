# Step-by-Step Learning Guide — DevOps Real-World Project on AWS

> **Who this is for:** Someone who knows **nothing** about Docker, Terraform, Kubernetes, Helm, AWS, or CI/CD. This guide takes you from zero to deploying and operating a real microservices application on AWS, one small step at a time.
>
> It is the *spine* of the course. Each section explains a concept in plain English, then points you at the exact module folder, the exact commands to run, how to check it worked, and how to clean up so you don't get a surprise AWS bill. Every command, file name, cluster name, and region below matches what is actually in this repository.

---

## How to use this guide

1. **Go in order, 01 → 21.** Each module builds on the previous one. Skipping ahead will break things.
2. **Read the "Concept" first, then do the "Steps."** Don't copy-paste blindly — understand *why* each step exists.
3. **Always run the "Clean up" step** when you finish a module for the day. AWS charges by the hour.
4. Open the `README.md` inside each module folder for the deep detail and screenshots — this guide is the map, the module READMEs are the territory.
5. When you see `<something-in-brackets>`, replace it with your own value.

**A note on the shell:** All commands below are written for a **Linux/Mac/EC2 bash** shell, because most of the hands-on work happens on an AWS Linux EC2 instance or inside the cluster. If you run them from Windows, use **Git Bash** or **WSL**, not PowerShell, so the syntax matches.

---

## Part 0 — The Big Picture (read once before starting)

### What problem does "DevOps" solve?

Imagine you wrote an online shop. It works on your laptop. Now you need it to:
- run on a server in the cloud, reliably, 24/7;
- survive a server crash without going down;
- handle a traffic spike (Black Friday) by adding capacity automatically;
- update to a new version with **zero downtime**;
- do all of the above **repeatably**, so a new environment can be built in minutes, not weeks.

**DevOps** is the set of tools and practices that make this possible. This course teaches the modern AWS toolchain for it.

### The application you'll deploy: "Retail Store"

A sample e-commerce site, intentionally built as **5 independent microservices** (separate small programs that talk to each other over the network) so you learn real-world patterns:

| Service | Language | Stores its data in |
|---|---|---|
| **UI** | Java (Spring Boot) | — (talks to the others) |
| **Catalog** | Go | MySQL |
| **Cart** | Java (Spring Boot) | DynamoDB |
| **Checkout** | Node.js | Redis |
| **Orders** | Java (Spring Boot) | PostgreSQL (+ a message queue, SQS) |

Early on you run these databases as containers; later you swap them for **AWS managed services** (RDS, DynamoDB, ElastiCache, SQS) — exactly what a real company does.

### The five tools you'll learn, and what each is *for*

| Tool | One-sentence job | Analogy |
|---|---|---|
| **Docker** | Packages an app + everything it needs into a portable "container." | A shipping container — same box runs anywhere. |
| **Terraform** | Creates cloud infrastructure (networks, servers, databases) from code. | A blueprint the cloud builds for you automatically. |
| **Kubernetes (EKS)** | Runs and manages your containers across many servers, healing and scaling them. | An air-traffic controller for containers. |
| **Helm** | Packages a whole Kubernetes app so it installs with one command. | An app store installer for Kubernetes. |
| **CI/CD (GitHub Actions + ArgoCD)** | Automatically builds and deploys your app when you push code. | A robot that ships your code for you. |

### How it all fits together (the end state of this course)

```
You push code to GitHub
        │
        ▼
GitHub Actions (CI) ──build──> Docker image ──push──> AWS ECR (image registry)
        │                                                   │
        └──updates image version in Git──┐                  │
                                          ▼                  │
                                  ArgoCD (CD) sees the change in Git
                                          │
                                          ▼
                       Kubernetes (AWS EKS) pulls the new image and deploys it
                                          │
        ┌─────────────────────────────────┼─────────────────────────────────┐
        ▼                                 ▼                                  ▼
   Karpenter adds EC2 nodes     HPA adds pod copies        ALB + Route53 serve users
   when capacity is needed      when CPU is high           at a friendly URL
        │
        ▼
   Observability (OpenTelemetry → CloudWatch / Prometheus / Grafana) shows you what's happening
```

You will build this **piece by piece**, bottom-up.

---

## Part 0.5 — One-Time Setup (do this before Module 02)

### 1. Accounts you need
- **AWS account** (a credit card is required; this course *will* cost a small amount of money — see cost note below).
- **Docker Hub account** (free) — for pushing Docker images in Modules 02–05.
- **GitHub account** (free) — for the CI/CD section (21).

### 2. Tools to install on your **local computer**

| Tool | Why | Install |
|---|---|---|
| AWS CLI v2 | Control AWS from the terminal | https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html |
| Terraform | Build infrastructure from code | https://developer.hashicorp.com/terraform/install |
| kubectl | Control Kubernetes | https://kubernetes.io/docs/tasks/tools/ |
| Helm | Install Kubernetes apps | https://helm.sh/docs/intro/install/ |
| Docker Desktop (optional) | Run containers locally | https://www.docker.com/products/docker-desktop |
| VS Code + "HashiCorp Terraform" extension | Edit `.tf` files comfortably | https://code.visualstudio.com |

Verify each is installed:
```bash
aws --version          # aws-cli/2.x
terraform -version     # Terraform v1.x
kubectl version --client
helm version
docker --version
```

### 3. Connect the AWS CLI to your account
In the AWS Console: **IAM → Users → your user → Security credentials → Create access key**. Then:
```bash
aws configure
# AWS Access Key ID     : <paste>
# AWS Secret Access Key : <paste>
# Default region name   : us-east-1
# Default output format : json
```
Test it:
```bash
aws sts get-caller-identity   # should print your account ID and user ARN
```

> **The whole course uses region `us-east-1`.** Stay in it to avoid confusion.

### 4. 💸 The Golden Rules of Cost (read twice)

AWS bills **per hour, whether you use the resource or not.** The expensive things in this course are:
- **EKS control plane** — ~$0.10/hour (~$73/month if left on).
- **NAT Gateway** — ~$0.045/hour + data (~$32/month each).
- **RDS / ElastiCache** databases — ~$30–100/month each.
- **EC2 worker nodes** — per instance.

**Rules:**
1. **Destroy everything at the end of each study session.** Every module has a "Clean up" step — do it.
2. **Never leave a cluster running overnight** unless you mean to.
3. After teardown, glance at the AWS **Billing** dashboard and the **EC2 / EKS / RDS / VPC (NAT Gateways) / Elastic IPs** consoles to confirm nothing is left running.
4. Set an **AWS Budget alert** (Billing → Budgets) for, say, $20, so you get an email if you forget.

---

## The Learning Roadmap

Modules group into **phases**. Each phase has a clear goal. The cluster/region/state-bucket are **reused** across modules, so a few key names recur everywhere:

| Recurring value | What it is |
|---|---|
| `retail-dev-eksdemo1` | The EKS cluster name (from Module 13 onward) |
| `us-east-1` | The AWS region used everywhere |
| `tfstate-dev-us-east-1-xxxxx` | Your Terraform state S3 bucket (you create it in 06_05) |
| `eksdemo1` | The base cluster name in Module 07 (becomes `retail-dev-eksdemo1` once the `retail`/`dev` prefixes are applied) |

| Phase | Modules | Goal |
|---|---|---|
| 1. Containers | 01–05 | Package and run the app as Docker containers |
| 2. Infrastructure as Code | 06–07 | Build a network + Kubernetes cluster with Terraform |
| 3. Kubernetes fundamentals | 08 | Learn Pods, Deployments, Services, ConfigMaps, StatefulSets |
| 4. Production Kubernetes on AWS | 09–12 | Secrets, storage, ingress, Helm |
| 5. The full app on a real cluster | 13–16 | Add-ons, AWS data plane, all 5 services, friendly DNS |
| 6. Autoscaling | 17–18 | Scale nodes (Karpenter) and pods (HPA) |
| 7. Helm for the whole app | 19 | One-command deploy of everything |
| 8. Observability | 20 | Traces, logs, metrics |
| 9. CI/CD (GitOps) | 21 | Automate build + deploy |

---

# PHASE 1 — Containers (Modules 01–05)

## Module 01 — Understand the Application
📁 `01-Project-Files/`

**Concept.** Before deploying anything, understand *what* you're deploying. Open `01-Project-Files/Code-Backup-1.3.0/` and look at `docker-compose.yaml` and `kubernetes.yaml`. This is the "Retail Store" app: **10 containers total** — the 5 microservices above plus their data stores (MariaDB, PostgreSQL, DynamoDB-local, Redis) and a RabbitMQ message broker.

**What to do.** Just read. The original app lives at `github.com/aws-containers/retail-store-sample-app`; the course uses a pinned fork so versions never break under you.

**Takeaway.** "Microservices" = many small programs, each with its own database, talking over the network. Everything that follows is about running these reliably.

---

## Module 02 — Docker on AWS EC2
📁 `02_Docker_Commands/` (sub-folders `02-01`, `02-02`, `02-03`)

**Concept.** **Docker** packages an app into an **image** (a frozen snapshot) which you run as a **container** (a live instance). You'll do this on a cloud Linux server (an **EC2 instance**) so the experience matches production.

### 02-01: Install Docker on EC2
1. In the AWS Console → **EC2 → Launch Instance**:
   - OS: **Amazon Linux 2023**; Type: **t3.large**; Storage: **30 GB**.
   - SSH key pair: create one and download the `.pem` file.
   - Security Group: open ports **22** (SSH), **80**, **8080**.
2. Connect to it:
   ```bash
   ssh -i your-key.pem ec2-user@<EC2-PUBLIC-IP>
   ```
3. Install Docker:
   ```bash
   sudo dnf update -y
   sudo dnf install docker -y
   sudo systemctl enable docker      # start on every reboot
   sudo systemctl start docker       # start now
   sudo usermod -aG docker ec2-user  # run docker without sudo
   ```
4. **Log out and back in** (so the group change applies), then verify:
   ```bash
   docker version
   docker run hello-world
   ```

### 02-02: Pull and run a pre-built image
```bash
docker pull stacksimplify/retail-store-sample-ui:1.0.0
# If Docker Hub rate-limits you, use the identical image from GitHub:
# docker pull ghcr.io/stacksimplify/retail-store-sample-ui:1.0.0

docker run --name myapp1 -p 8888:8080 -d stacksimplify/retail-store-sample-ui:1.0.0
docker ps                                 # see it running
# Browse to http://<EC2-PUBLIC-IP>:8888
docker exec -it myapp1 /bin/sh            # open a shell inside the container
docker logs myapp1                        # see its output
docker stop myapp1 && docker rm myapp1    # stop and remove
```
- `-p 8888:8080` maps **your** port 8888 to the container's port 8080.
- `-d` = run detached (in the background).

### 02-03: Build your own image and push it to Docker Hub
```bash
docker login
wget https://github.com/aws-containers/retail-store-sample-app/archive/refs/tags/v1.2.4.zip
unzip v1.2.4.zip
cd retail-store-sample-app-1.2.4/src/ui
docker build -t retail-store-sample-ui:2.0.0 .
docker run --name myapp1-v2 -p 8889:8080 -d retail-store-sample-ui:2.0.0
# Tag with YOUR Docker Hub username and push:
docker tag retail-store-sample-ui:2.0.0 <your-dockerhub-user>/retail-store-sample-ui:2.0.0
docker push <your-dockerhub-user>/retail-store-sample-ui:2.0.0
```

**Clean up (important):** Stop the EC2 instance from the AWS Console when done (Instances → Stop or Terminate). Inside it you can also run `docker system prune -a --volumes -f` to reclaim space.

**Watch out for:** the `docker prune -a` command deletes *all* unused images/containers — fine here, dangerous in real life.

---

## Module 03 — Dockerfiles
📁 `03_Docker_Files/`

**Concept.** A **Dockerfile** is the recipe Docker follows to build an image. This module shows a **multi-stage build**: stage 1 compiles the Java code (needs Maven + JDK), stage 2 copies *only* the finished `.jar` into a tiny runtime image. Result: a small, secure, fast image that runs as a non-root user.

```bash
cd retail-store-sample-app-1.2.4/src/ui
cat Dockerfile                       # read the recipe
docker build -t retail-ui:9.0.0 .
docker run -d --name retail-ui -p 8080:8080 retail-ui:9.0.0
# Browse http://<EC2-PUBLIC-IP>:8080/actuator/health  → should say UP
```
Prove the build tools aren't in the final image:
```bash
docker exec -it retail-ui sh
  which mvn   # "command not found" — good, Maven was left behind in stage 1
  ls /src     # "No such file" — source code isn't shipped
  exit
```
Notice caching: the first build takes ~110s, a no-change rebuild ~0.3s, and `docker build --no-cache` forces a full rebuild.

**Clean up:** `docker stop retail-ui && docker rm retail-ui && docker rmi retail-ui:9.0.0`

---

## Module 04 — Docker Compose
📁 `04_Docker_Compose/`

**Concept.** Running 10 `docker run` commands by hand is painful. **Docker Compose** defines all services in one `docker-compose.yaml` and starts them together, on a shared network, with health checks and startup ordering.

```bash
mkdir demo-compose && cd demo-compose
wget https://github.com/aws-containers/retail-store-sample-app/releases/download/v1.3.0/docker-compose.yaml
export DB_PASSWORD='mydbkalyan101'      # the compose file reads this
docker compose up -d                    # start all 10 services in the background
docker compose ps                       # status of everything
# Browse http://<EC2-PUBLIC-IP>:8888  → the full shop, end to end
```
Useful commands:
```bash
docker compose logs -f checkout         # follow one service's logs
docker compose exec ui sh               # shell into a service
docker compose stop orders              # stop just one service
docker compose restart cart
docker compose down                     # stop & remove everything
```
**Try it:** edit the `ui` service's environment and set `RETAIL_UI_THEME=green`, then `docker compose up -d --force-recreate ui`. The shop changes colour — your first taste of configuration-as-environment.

**Clean up:** `docker compose down` then `docker system prune -a --volumes -f`.

**Key idea:** services find each other **by name** (`http://catalog:8080`) because they share a Docker network. This name-based discovery reappears in Kubernetes.

---

## Module 05 — Docker Buildx (multi-architecture images)
📁 `05_Docker_Buildx/`

**Concept.** Laptops/most servers are **AMD64**; AWS's cheap **Graviton** chips are **ARM64**. An AMD64-only image won't run on ARM64. **Buildx** builds one image tag that works on *both*.

```bash
docker run --privileged --rm tonistiigi/binfmt --install all   # enable cross-arch emulation
docker buildx create --name multiarch --driver docker-container --use
docker buildx inspect --bootstrap

export IMAGE="<your-dockerhub-user>/retail-ui-multiarch:1.0.0"
docker login
docker buildx build --platform linux/amd64,linux/arm64 -t "$IMAGE" --push .
docker buildx imagetools inspect "$IMAGE"    # shows BOTH amd64 and arm64
```
**Watch out for:** building ARM64 on an AMD64 host uses emulation and is **slow** (30+ minutes). Be patient; don't interrupt it.

---

# PHASE 2 — Infrastructure as Code (Modules 06–07)

## Module 06 — Terraform Basics
📁 `06_Terraform_Basics/` (sub-folders `06_01` … `06_07`)

**Concept.** Instead of clicking in the AWS Console, you describe infrastructure in `.tf` files and Terraform creates it. The four commands you'll use constantly:
```bash
terraform init      # download the AWS plugin (once per folder)
terraform validate  # syntax check
terraform plan      # PREVIEW what will change — makes no changes
terraform apply -auto-approve   # actually build it
terraform destroy -auto-approve # tear it down
```

Work through the sub-modules in order — each adds one new idea:

| Sub | What it teaches | Key files |
|---|---|---|
| **06_01** | Install Terraform + AWS CLI | (install only) |
| **06_02** | The 3 building blocks: `terraform{}`, `provider{}`, `resource{}` — builds a random-named S3 bucket | `c1-versions.tf`, `c2-s3bucket.tf`, `c3-outputs.tf` |
| **06_03** | A real **VPC** (network) with public/private subnets, NAT, route tables; **local** state | `c1-versions.tf` … `c5-outputs.tf` |
| **06_04** | **Variable precedence** — defaults < env vars < `terraform.tfvars` < `*.auto.tfvars` < `-var-file` < `-var` | `var-files/*.tfvars` |
| **06_05** | Create an **S3 bucket for remote state** (so state is shared & locked) | `c3-s3bucket.tf` |
| **06_06** | Point the VPC project at that **remote backend** | `c1-versions.tf` (backend block) |
| **06_07** | Wrap the VPC in a reusable **module** | `modules/vpc/` |

Typical run (inside any sub-module's `terraform-manifests/` folder):
```bash
cd 06_Terraform_Basics/06_02_terraform_foundation/terraform-manifests
terraform init && terraform apply -auto-approve
# ... look at what it made ...
terraform destroy -auto-approve
```

**Two concepts to really grasp here:**
- **State** — Terraform records what it built in a `terraform.tfstate` file. In `06_05`/`06_06` you move this file to an **S3 bucket** so a team can share it and it can't be corrupted by two people at once. **Write down your bucket name** (e.g. `tfstate-dev-us-east-1-jpjtof`) — every later module references it.
- **A VPC** is your private network in AWS: public subnets (for load balancers) + private subnets (for servers), connected to the internet via an Internet Gateway and NAT Gateway.

**Clean up:** `terraform destroy -auto-approve` in each folder. (The NAT Gateway in the VPC modules costs money — don't leave it.)

---

## Module 07 — EKS Cluster with Terraform
📁 `07_Terraform_EKS_Cluster/` → `01_VPC_terraform-manifests/`, `02_EKS_terraform-manifests/`

**Concept.** **EKS** is AWS's managed Kubernetes. AWS runs the cluster "brain" (control plane); you supply the worker servers (a **node group**). You build it in two stages: **VPC first, then EKS inside it**. The EKS project reads the VPC's outputs from the shared S3 state (a "remote state datasource").

```bash
# Stage 1 — the network
cd 07_Terraform_EKS_Cluster/01_VPC_terraform-manifests
terraform init && terraform apply -auto-approve

# Stage 2 — the cluster (reads the VPC's state from S3)
cd ../02_EKS_terraform-manifests
terraform init && terraform apply -auto-approve

# Connect kubectl to your new cluster
aws eks update-kubeconfig --name eksdemo1 --region us-east-1
kubectl get nodes                 # your worker nodes, should be "Ready"
kubectl get pods -n kube-system   # the cluster's own system pods
```
Defaults baked into `terraform.tfvars`: cluster `eksdemo1`, Kubernetes `1.34`, nodes `t3.small` × (desired 3, min 1, max 6), in **private subnets**.

**Clean up — ORDER MATTERS (reverse of creation):**
```bash
cd 02_EKS_terraform-manifests && terraform destroy -auto-approve
cd ../01_VPC_terraform-manifests && terraform destroy -auto-approve
```
> Destroying the VPC before the EKS cluster will hang/fail because the cluster still uses the network. **Always EKS → then VPC.**

**Watch out for:** This is the first *expensive* module. Control plane + nodes + NAT ≈ several dollars/day. Destroy when done.

---

# PHASE 3 — Kubernetes Fundamentals (Module 08)

📁 `08_Kubernetes_Foundation/` — needs a running EKS cluster (from Module 07).

**Concept.** Kubernetes runs your containers for you. You declare *what you want* in YAML, run `kubectl apply -f file.yaml`, and Kubernetes makes reality match. Core objects, smallest to largest:

```
Container → Pod → ReplicaSet → Deployment
                                    │
                              exposed by a Service
                              configured by a ConfigMap / Secret
```

### 08-01 Pods — the smallest unit (one or more containers)
```bash
kubectl apply -f 01_catalog_pod.yaml
kubectl get pods
kubectl describe pod catalog-pod          # details + events (your #1 debugging tool)
kubectl logs -f catalog-pod
kubectl port-forward pod/catalog-pod 7080:8080   # reach it from your machine
# Browse http://localhost:7080/health and /catalog/products
kubectl exec -it catalog-pod -- sh        # shell inside
kubectl delete pod catalog-pod
```

### 08-02 Deployments — keep N copies alive, do rolling updates & rollbacks
```bash
kubectl apply -f 01_catalog_deployment.yaml
kubectl scale deployment catalog --replicas=3            # scale out
kubectl set image deployment/catalog catalog=public.ecr.aws/aws-containers/retail-store-sample-catalog:1.3.0  # rolling update
kubectl rollout status deployment/catalog
kubectl rollout undo deployment/catalog                  # roll back
```
Relationship: a **Deployment** manages a **ReplicaSet**, which manages **Pods**. If a pod dies, the ReplicaSet replaces it.

### 08-03 Services — a stable address + load balancing for pods
```bash
kubectl apply -f 02_catalog_clusterip_service.yaml
kubectl get svc
# from inside the cluster, the service is reachable as: catalog-service:8080
```
Types: **ClusterIP** (internal only — the default), **NodePort**, **LoadBalancer** (public AWS LB). Pods are matched to a Service by **labels**.

### 08-04 ConfigMaps — externalise non-secret configuration
```bash
kubectl apply -f catalog_k8s_manifests/
kubectl exec -it <catalog-pod> -- env      # see the injected variables
```

### 08-05 StatefulSets — for stateful apps (a MySQL database)
StatefulSets give pods **stable names** (`catalog-mysql-0`, `-1`, …) and a **headless Service** for stable DNS. Scaling happens in **order**.
```bash
kubectl apply -f catalog_k8s_manifests
kubectl get statefulsets
kubectl scale statefulset catalog-mysql --replicas=3   # created 0→1→2 in order
# connect a temporary MySQL client:
kubectl run mysql-client --rm -it --image=mysql:8.0 --restart=Never -- mysql -h catalog-mysql -u catalog_user -p
# password: kalyandb101
```
> ⚠️ This module uses an **emptyDir** volume, so **data is lost** when the pod dies. Module 10 fixes that with real storage.

**Clean up after each sub-module:** `kubectl delete -f <the-folder-or-file>`.

---

# PHASE 4 — Production Kubernetes on AWS (Modules 09–12)

These add the things you need for a real workload: secure secrets, persistent storage, public access, and packaging. They all assume a running cluster.

## Module 09 — Secrets
📁 `09_Kubernetes_Secrets/`

**Concept.** Passwords don't belong in ConfigMaps or images. You'll progress from basic Kubernetes Secrets to the **production-grade** pattern: store secrets in **AWS Secrets Manager** and let pods fetch them via the **Secrets Store CSI Driver**, authenticating with **EKS Pod Identity** (no static AWS keys anywhere).

| Sub | What it adds |
|---|---|
| **09-01** | Native K8s `Secret` (base64 — *obfuscation, not encryption*). `kubectl apply -f catalog_k8s_manifests` |
| **09-02** | **EKS Pod Identity** — lets a pod assume an IAM role. Demo: an AWS-CLI pod can't `aws s3 ls` until you attach a role, then it can. |
| **09-03** | Install the **Secrets Store CSI Driver** + **AWS provider (ASCP)** via Helm; create the IAM role + Pod Identity association. |
| **09-04** | Put real MySQL creds in **AWS Secrets Manager**, define a **SecretProviderClass**, mount them at `/mnt/secrets-store`. No plaintext creds in the cluster. |

Representative commands (09-03):
```bash
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm repo add aws-secrets-manager https://aws.github.io/secrets-store-csi-driver-provider-aws
helm repo update
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver -n kube-system
helm install secrets-provider-aws aws-secrets-manager/secrets-store-csi-driver-provider-aws \
  -n kube-system --set secrets-store-csi-driver.install=false
```
And 09-04 creates the secret:
```bash
aws secretsmanager create-secret --name catalog-db-secret-1 --region us-east-1 \
  --secret-string '{"MYSQL_USER":"mydbadmin","MYSQL_PASSWORD":"kalyandb101"}'
```

**Big idea — Pod Identity:** a Kubernetes **ServiceAccount** is linked to an **AWS IAM role**; pods using that ServiceAccount automatically get temporary AWS credentials. This is how every AWS-integrated component (storage driver, load balancer controller, etc.) authenticates from here on. (After creating a new association, **restart the pod** so it picks it up.)

## Module 10 — Storage
📁 `10_Kubernetes_Storage/`

**Concept.** Real databases need data that survives pod restarts. The **EBS CSI Driver** lets Kubernetes create AWS **EBS** disks on demand via a `PersistentVolumeClaim`.

```bash
# 10-01 install the driver as an EKS add-on (after creating its IAM role + Pod Identity association)
aws eks create-addon --cluster-name retail-dev-eksdemo1 --addon-name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::<ACCOUNT_ID>:role/AmazonEKS_EBS_CSI_DriverRole_retail-dev-eksdemo1

# 10-02 give MySQL a real EBS-backed volume (StorageClass + volumeClaimTemplates)
kubectl apply -f 02_catalog_k8s_manifests/
kubectl get sc,pvc,pv,pods
# delete the MySQL pod, and after it restarts the data is STILL THERE — true persistence

# 10-03 better still: use a managed RDS MySQL database, reached via an ExternalName Service
```
**Watch out for:** an EBS volume keeps costing money until you delete its **PVC** (`kubectl delete pvc data-catalog-mysql-0`). In 10-03 you also delete the RDS instance from the console.

## Module 11 — Ingress (public access)
📁 `11_Kubernetes_Ingress/`

**Concept.** An **Ingress** is a smart HTTP router. The **AWS Load Balancer Controller** turns an Ingress object into a real **ALB** (Application Load Balancer) that routes internet traffic to your services.

```bash
# 11-01 install the controller (IAM policy from AWS, role, Pod Identity association, then Helm)
helm repo add eks https://aws.github.io/eks-charts && helm repo update
VPC_ID=$(aws eks describe-cluster --name retail-dev-eksdemo1 --query "cluster.resourcesVpcConfig.vpcId" --output text)
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=retail-dev-eksdemo1 --set region=us-east-1 --set vpcId=$VPC_ID \
  --set serviceAccount.create=true --set serviceAccount.name=aws-load-balancer-controller

# 11-02 HTTP ingress — deploy all 5 services + an Ingress
kubectl apply -R -f http_retail_store_k8s_manifests/
kubectl get ingress -A          # wait 2–6 min for the ALB, then browse its DNS name

# 11-03 HTTPS ingress — needs an ACM certificate + a Route53 domain you own
```
**Watch out for:** ALB creation takes **2–6 minutes** — be patient. HTTPS (11-03) **requires a real registered domain** in Route 53; if you don't have one, just read it.

## Module 12 — Helm
📁 `12_Helm/`

**Concept.** **Helm** is the package manager for Kubernetes. A **chart** = `Chart.yaml` (metadata) + `values.yaml` (defaults) + `templates/` (parameterised YAML). One command installs a whole app.

| Sub | What it teaches |
|---|---|
| **12-01** | install / upgrade / rollback / uninstall a chart from a registry (OCI) |
| **12-02** | override settings with a custom `values-ui.yaml` (enables ALB ingress) |
| **12-03** | `helm pull --untar`, `helm lint`, `helm template` to inspect/render a chart locally |
| **12-04** | `helm package` + `helm push` your own chart to **private ECR** |
| **12-05** | deploy **all 5 services** with Helm via `./install-retail-apps.sh` |

```bash
# 12-01
aws ecr-public get-login-password --region us-east-1 | helm registry login -u AWS --password-stdin public.ecr.aws
helm install ui oci://public.ecr.aws/aws-containers/retail-store-sample-ui-chart --version 1.0.0
helm list
helm upgrade ui oci://public.ecr.aws/aws-containers/retail-store-sample-ui-chart --version 1.3.0 --set app.theme=green
helm rollback ui 1
helm uninstall ui
```
**Watch out for:** the ECR-public login token lasts **12 hours** — re-run the login if `helm install` later fails with auth errors. Service names may differ from the release name — check with `kubectl get svc`.

---

# PHASE 5 — The Full App on a Real Cluster (Modules 13–16)

This is where it becomes a *real system*. From here on the cluster is named **`retail-dev-eksdemo1`**.

## Module 13 — EKS Cluster with Add-Ons (built by Terraform)
📁 `13_Terraform_EKS_Cluster_with_AddOns/`

**Concept.** Modules 09–11 installed the Pod Identity agent, EBS CSI driver, Load Balancer Controller and Secrets Store CSI driver **by hand**. Real teams bake all of that into Terraform so a complete, production-ready cluster comes up with one workflow. That's this module.

```bash
# Stage 1: VPC
cd 13_Terraform_EKS_Cluster_with_AddOns/01_VPC_terraform-manifests
terraform init && terraform apply -auto-approve
# Stage 2: EKS + all add-ons (files c11…c16 install the add-ons)
cd ../02_EKS_terraform-manifests_with_addons
terraform init && terraform apply -auto-approve

aws eks update-kubeconfig --name retail-dev-eksdemo1 --region us-east-1
kubectl get pods -n kube-system        # see all the add-on pods Running
aws eks list-addons --cluster-name retail-dev-eksdemo1
```
There are also convenience scripts: `./create-cluster.sh` and `./destroy-cluster.sh`.

**Clean up:** `./destroy-cluster.sh` (or destroy EKS folder then VPC folder).

> **From here, keep this cluster running** while you work through 14–16 in one session, then tear it all down at the end.

## Module 14 — The Full Retail Store on the AWS Data Plane
📁 `14_RetailStore_Microservices_with_AWS_Data_Plane/`

**Concept — "data plane":** the managed AWS data stores the app uses instead of in-cluster containers. **14_01** (Terraform) creates them; **14_02** (kubectl) wires the microservices to them with zero stored credentials (Pod Identity + Secrets Manager).

```bash
# 14_01 — provision RDS MySQL, RDS PostgreSQL, DynamoDB, ElastiCache Redis, SQS, Secrets, IAM
cd .../14_01_RetailStore_AWS_Data_Plane/03_AWS_Data_Plane_terraform-manifests
terraform init && terraform apply -auto-approve     # or ./create-aws-dataplane.sh

# 14_02 — deploy the 5 services and connect them
cd .../14_02_Microservices_with_AWS_Data_Plane/RetailStore_k8s_manifests_with_Data_Plane
kubectl apply -f 01_secretproviderclass/
kubectl apply -f 02_RetailStore_Microservices/05_ui/ && kubectl apply -f 03_ingress/
kubectl apply -f 02_RetailStore_Microservices/01_catalog/   # → RDS MySQL
kubectl apply -f 02_RetailStore_Microservices/02_cart/      # → DynamoDB
kubectl apply -f 02_RetailStore_Microservices/03_checkout/  # → ElastiCache Redis
kubectl apply -f 02_RetailStore_Microservices/04_orders/    # → PostgreSQL + SQS
kubectl get pods && kubectl get ingress
# Browse the ALB URL → full shop on managed AWS services
```
Each service folder follows the same pattern: `service_account` → `configmap` → `deployment` → `clusterip_service` (+ an `externalname_service` for databases). There are also handy **verification pods** under `04_Verification_Pods/` to test each backend.

**Watch out for:** this module runs **several paid databases at once** (RDS ×2, ElastiCache, etc.) — easily a few dollars/day. Run `./delete-aws-dataplane.sh` (then `terraform destroy`) as soon as you're done.

## Modules 15 & 16 — ExternalDNS (friendly URLs)
📁 `15_Terraform_EKS_Cluster_ExternalDNS/`, `16_RetailStore_Microservices_ExternalDNS/`

**Concept.** Nobody wants to type `k8s-default-ui-b67f...elb.amazonaws.com`. **ExternalDNS** watches your Ingress objects and automatically creates **Route 53** DNS records like `retailstore1.stacksimplify.com`.

- **15** installs ExternalDNS as an EKS add-on via Terraform (files `c17-01/02/03`).
- **16** deploys the app and adds an annotation to the Ingress:
  ```yaml
  external-dns.alpha.kubernetes.io/hostname: retailstore1.stacksimplify.com
  ```
  ExternalDNS then creates the Route 53 record automatically; deleting the Ingress removes it.

**Watch out for:** requires a **real domain + Route 53 hosted zone**. No domain → read along and understand the flow.

---

# PHASE 6 — Autoscaling (Modules 17–18)

Two *different* kinds of scaling. Learn the distinction:
- **Karpenter** scales **nodes** (adds/removes EC2 servers when pods can't fit).
- **HPA** scales **pods** (adds/removes copies of a service when CPU/memory is high).

## Module 17 — Karpenter (node autoscaling)
📁 `17_Autoscaling_Karpenter/`

**Concept.** When a pod can't be scheduled (no node has room), **Karpenter launches a right-sized EC2 instance in ~30–60 seconds**, and removes nodes when they're underused. It supports **Spot** instances (up to ~70% cheaper) and handles Spot interruptions gracefully.

```bash
# 17_01 install: VPC → EKS → Karpenter Terraform → then the K8s objects
cd 17_01_Karpenter_Install/04_KARPENTER_k8s-manifests
kubectl apply -f 01_ec2nodeclass.yaml      # what kind of nodes Karpenter may make
kubectl apply -f 02_nodepool_ondemand.yaml
kubectl apply -f 03_nodepool_spot.yaml

# 17_02 watch it scale on demand
kubectl apply -f .../On-demand_autoscaling_test.yaml
kubectl get nodeclaims -w                  # new nodes appear as pods need them
kubectl scale deploy/karpenter-autoscale-demo-ondemand --replicas=10   # more nodes
kubectl scale deploy/karpenter-autoscale-demo-ondemand --replicas=2    # nodes consolidate away

# 17_03 prove Spot nodes are used
kubectl get nodes --selector=karpenter.sh/capacity-type=spot

# 17_04 simulate a Spot interruption and watch zero-downtime migration (PodDisruptionBudget)
```
**Clean up — extra step:** delete NodePools **before** Terraform destroy, or use the script:
```bash
kubectl delete nodepools --all
./destroy-cluster-with-karpenter.sh
```

## Module 18 — Horizontal Pod Autoscaler (HPA)
📁 `18_Autoscaling_HPA/`

**Concept.** HPA needs the **Metrics Server** to read CPU/memory. Then an `HorizontalPodAutoscaler` keeps each service between `minReplicas` and `maxReplicas`, targeting (e.g.) 80% CPU. This module also adds **PodDisruptionBudgets** (keep N pods up during disruptions) and **TopologySpreadConstraints** (spread pods across AZs/nodes).

```bash
# 1) install Metrics Server (Terraform)
cd 01_Metrics_Server_terraform-manifests && terraform init && terraform apply -auto-approve
# 2) data plane + microservices (as in Module 14)
# 3) HPA + PDB
kubectl apply -f 04_HPA/   && kubectl get hpa
kubectl apply -f 05_PDB/   && kubectl get pdb
./check-topology.sh        # see pods spread 1-per-AZ
```
`03_..._ScheduleAnyway` vs `04_..._DoNotSchedule` show the two spread policies (best-effort vs strict).

---

# PHASE 7 — Helm for the Whole App (Module 19)

📁 `19_Helm_RetailStore_AWS_Dataplane/`

**Concept.** Combine everything: spin up the Karpenter cluster, the AWS data plane, then deploy **all 5 services via Helm** with custom values files — `v1.0.0` (config via env vars) and `v2.0.0` (secrets via AWS Secrets Manager). This is the clean, repeatable way a team ships the app.

```bash
cd 19_Helm_RetailStore_AWS_Dataplane/01_EKS_Cluster_Environment && ./create-cluster-with-karpenter.sh
cd ../02_RetailStore_AWS_Dataplane && ./create-aws-dataplane.sh
cd ../03_RetailStore_Helm_with_Data_Plane/02_retailstore_values_HELM_aws_dataplane
./03-v1.0.0-install-remote-helm-charts.sh    # one command, whole app
helm list && kubectl get pods && kubectl get ingress
./05-v2.0.0-install-remote-helm-charts.sh    # upgrade to the Secrets-Manager version
```
**Clean up (full stack):**
```bash
./01-uninstall-retail-apps.sh
cd ../../02_RetailStore_AWS_Dataplane && ./delete-aws-dataplane.sh
cd ../01_EKS_Cluster_Environment && ./destroy-cluster-with-karpenter.sh
```

---

# PHASE 8 — Observability (Module 20)

📁 `20_Observability_OpenTelemetry/`

**Concept.** "Observability" = understanding a running system from the outside via three signals:

| Signal | Answers | Goes to |
|---|---|---|
| **Traces** | "How did one request travel through all 5 services?" | AWS X-Ray / CloudWatch Application Signals |
| **Logs** | "What did the app print, and when?" | CloudWatch Logs |
| **Metrics** | "How much CPU/memory/requests over time?" | Amazon Managed Prometheus → Grafana |

The **ADOT collector** (AWS Distro for OpenTelemetry) gathers all three from your pods and ships them to AWS.

```bash
# 20_01 build an EKS+Karpenter cluster with ADOT, cert-manager, Prometheus, Grafana (Terraform)
./create-cluster-with-karpenter-and-opentelemetry.sh

# 20_02 Traces → X-Ray
kubectl apply -f 01_OpenTelemetry_Traces/01_adot_collector_traces.yaml
kubectl apply -f 01_OpenTelemetry_Traces/02_adot_instrumentation_traces.yaml
./restart-retailapp.sh        # apps pick up tracing, then browse the shop and watch traces appear

# 20_03 Logs → CloudWatch (ADOT runs as a DaemonSet reading /var/log/pods)
kubectl apply -f 01_OpenTelemetry_Logs/01_adot_collector_logs.yaml

# 20_04 Metrics → Amazon Managed Prometheus → Grafana
#  - paste your AMP "remote write" endpoint into the collector YAML, then apply it
#  - in Grafana, add the AMP data source and import dashboard ID 15661
```
**Watch out for:** **Amazon Managed Grafana costs ~$9 per assigned user**, and AMP/CloudWatch charge for ingested data. The traces/logs configs deliberately **filter out health-check noise** to cut cost — keep that. Grafana needs **AWS IAM Identity Center** enabled.

---

# PHASE 9 — CI/CD with GitHub Actions + ArgoCD (Module 21)

📁 `21_DevOps_CICD/` — uses a dedicated repo: `github.com/stacksimplify/aws-devops-github-actions-ecr-argocd3`

**Concept — GitOps.** Git is the single source of truth. **CI** (GitHub Actions) builds the image and records the new version in Git; **CD** (ArgoCD) notices the Git change and deploys it. No human runs `kubectl` or `helm` to ship.

```
push code → GitHub Actions builds image → pushes to ECR (tags: latest + sha-<commit>)
          → updates chart/values-ui.yaml in Git
          → ArgoCD (polls every ~3 min) sees the change → deploys to EKS
```

### 21-01 CI — GitHub Actions → ECR
1. Create the image registry:
   ```bash
   aws ecr create-repository --repository-name retail-store/ui --region us-east-1
   ```
2. Create a **GitHub OIDC IAM role** so Actions can push to ECR **without storing AWS keys** (trust policy scoped to your repo; attach `AmazonEC2ContainerRegistryPowerUser`; create the OIDC provider for `token.actions.githubusercontent.com`).
3. Add `.github/workflows/build-push-ui.yaml` (provided): it checks out code, assumes the role via OIDC, logs in to ECR, builds + pushes **two tags** (`latest` for convenience, `sha-<commit>` — immutable — for production), then `sed`-updates `chart/values-ui.yaml` and commits it back.
4. Push a change under `src/ui/src/**` to trigger it; watch the **Actions** tab; verify with `aws ecr describe-images --repository-name retail-store/ui`.

> **Why two tags?** `latest` moves and is unsafe for prod; `sha-<commit>` ties each deploy to an exact commit → perfect traceability and safe rollbacks.

### 21-02 CD — install ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl port-forward svc/argocd-server -n argocd 8080:443   # open https://localhost:8080
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 --decode && echo
# log in as admin / <that password>
```

### 21-03 CD — point ArgoCD at your Git repo
Apply an **Application** manifest (`argocd-manifests/application-ui.yaml`) that says: *watch this repo/branch/path, render the Helm chart with `values-ui.yaml`, deploy to the cluster, and keep them in sync* (`prune: true`, `selfHeal: true`). For a private repo, register it first with `argocd repo add ... --username ... --password <PAT>`.
```bash
kubectl apply -f argocd-manifests/application-ui.yaml
kubectl get applications -n argocd        # NAME ui  SYNC Synced  HEALTH Healthy
```

### 21-04 Full flow test
Deploy the backend services, then change the UI version and push:
```bash
./update-ui-home-html.sh V904 && ./git-push.sh V904
```
Watch the chain: Actions builds & pushes → updates Git → ArgoCD syncs within ~3 min → new pods roll out → the live site shows **V904**. Try a **rollback** from ArgoCD's History tab — it reverts the cluster to a previous Git state. That's GitOps: **manual cluster changes get auto-reverted; rollbacks are just Git.**

---

## 🧹 Cost Management & Full Teardown Checklist

Run the relevant cleanup after **every** session. To be sure nothing lingers, check these AWS consoles:

```bash
# Helm apps
helm uninstall <release> -n <namespace>        # or the provided ./*uninstall*.sh

# Kubernetes extras
kubectl delete nodepools --all                 # Karpenter (before destroying EKS!)
kubectl delete application ui -n argocd

# Terraform — ALWAYS in reverse creation order:
#   data plane → karpenter → EKS → VPC
terraform destroy -auto-approve                # in each folder, deepest first
# or the provided ./destroy-*.sh scripts
```

Manual console checks (these are the ones that quietly cost money):
- **EKS** → no clusters left
- **EC2 → Instances** → none running (also check **Elastic IPs** — release any unattached)
- **VPC → NAT Gateways** → none (≈$32/mo each)
- **RDS → Databases** and **RDS → Subnet groups** → none
- **ElastiCache** → no clusters
- **EBS → Volumes** → none "available" (orphaned by undeleted PVCs)
- **Secrets Manager**, **DynamoDB**, **SQS**, **Managed Grafana/Prometheus**, **Load Balancers** → clean

---

## 🛠️ Common Errors & Fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| `kubectl` "connection refused" / "Unauthorized" | kubeconfig not pointed at the cluster | `aws eks update-kubeconfig --name retail-dev-eksdemo1 --region us-east-1` |
| Pod stuck `Pending` | No node has room | Karpenter will add a node (17); or check `kubectl describe pod` events |
| Pod `CrashLoopBackOff` | App can't reach its DB / bad config | `kubectl logs <pod>` and `kubectl describe pod <pod>` |
| `terraform destroy` hangs on VPC | EKS/ENIs still using it | Destroy EKS folder first, then VPC |
| Helm install fails with ECR auth error | 12-hour ECR-public token expired | re-run the `helm registry login` command |
| Ingress has no ADDRESS | ALB still provisioning | wait 2–6 min; check Load Balancer Controller logs in `kube-system` |
| Pod can't access AWS (S3/Secrets/etc.) | Missing/just-created Pod Identity association | create the association, then **restart the pod** |
| GitHub Actions can't push to ECR | OIDC trust policy repo name mismatch / role policy missing | check repo name (case-sensitive) and `AmazonEC2ContainerRegistryPowerUser` is attached |

---

## 📖 Mini-Glossary

- **Image / Container** — a packaged app / a running instance of it.
- **Registry (Docker Hub, ECR)** — where images are stored.
- **EC2** — a virtual server in AWS. **EBS** — a virtual disk for it.
- **VPC / Subnet / NAT Gateway** — your private cloud network / its segments / its internet exit for private servers.
- **EKS** — AWS-managed Kubernetes. **Node** — a worker server in the cluster.
- **Pod** — smallest K8s unit (1+ containers). **Deployment** — keeps N pods running. **Service** — stable address for pods. **Ingress** — HTTP router → ALB.
- **ConfigMap / Secret** — non-secret / secret configuration.
- **StatefulSet / PVC** — stable-identity pods (databases) / a storage request.
- **IAM role / Pod Identity** — AWS permissions / how a pod safely assumes a role.
- **Helm chart / values** — a K8s app package / its settings.
- **Terraform / state** — infrastructure-as-code / its record of what exists.
- **Karpenter (nodes) vs HPA (pods)** — the two autoscalers.
- **ADOT / traces, logs, metrics** — the observability collector / the three signals.
- **CI / CD / GitOps** — auto-build / auto-deploy / Git as the source of truth (ArgoCD).

---

*Follow 02 → 21 in order. Read each module's own `README.md` for full depth and screenshots. Above all: **build it, watch it work, then destroy it.** Repetition + teardown discipline is how this sticks (and how your bill stays small).*
