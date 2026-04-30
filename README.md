# Configuration Management with Ansible

A hands-on DevOps project demonstrating infrastructure automation, configuration management, and application deployment across cloud environments using Ansible. This project covers progressively advanced patterns — from simple playbooks to role-based automation — targeting real-world AWS EC2 and Kubernetes environments.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture Diagram](#architecture-diagram)
- [Technologies Used](#technologies-used)
- [Infrastructure & Inventory](#infrastructure--inventory)
- [Project Deliverables](#project-deliverables)
  - [1. Nexus Repository Manager Deployment](#1-nexus-repository-manager-deployment)
  - [2. Node.js Application Deployment](#2-nodejs-application-deployment)
  - [3. Docker & Docker Compose Automation](#3-docker--docker-compose-automation)
  - [4. Role-Based Automation](#4-role-based-automation)
  - [5. EKS Cluster Provisioning + Kubernetes Application Deployment](#5-eks-cluster-provisioning--kubernetes-application-deployment)
- [Ansible Roles](#ansible-roles)
- [Key Outcomes & Skills Demonstrated](#key-outcomes--skills-demonstrated)
- [Project Structure](#project-structure)

---

## Project Overview

This project automates the provisioning, configuration, and deployment of applications across cloud infrastructure using **Ansible** and **Terraform**. It demonstrates a progression from manual shell scripting to fully automated, idempotent, role-based infrastructure-as-code — a core competency in modern DevOps practice.

The project spans **two cloud providers** (AWS and DigitalOcean) and uses **two separate Terraform projects** to provision infrastructure before Ansible manages configuration and deployments:

- **`terraform-learn/`** — provisions an AWS EC2 instance (`ca-central-1`) with Docker pre-installed via user_data; Ansible deploys the Docker Compose stack on top
- **`Terraform/` (EKS)** — provisions a full VPC + managed EKS cluster (`eu-central-1`); Ansible deploys Kubernetes workloads on top

DigitalOcean Droplets (Nexus, Node.js) are targeted directly via static inventory without Terraform provisioning.

---

## Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                              CONTROL NODE  (Local Machine)                               │
│                                                                                          │
│  ┌──────────────────────────────┐      ┌──────────────────────────────────────────────┐ │
│  │     TERRAFORM  (Stage 1)     │      │           ANSIBLE  (Stage 2)                 │ │
│  │──────────────────────────────│      │──────────────────────────────────────────────│ │
│  │ terraform-learn/             │      │ ansible.cfg  (ec2-user, id_rsa, aws_ec2)     │ │
│  │  ├── providers.tf            │      │                                              │ │
│  │  │    └─ AWS ~> 6.0          │      │ Inventory                                    │ │
│  │  ├── main.tf                 │      │  ├── hosts  (static: nexus, docker servers)  │ │
│  │  │    ├── VPC + subnet       │      │  └── inventory_aws_ec2.yaml (dynamic: tags)  │ │
│  │  │    ├── Internet Gateway   │      │                                              │ │
│  │  │    ├── Security Group     │      │ Playbooks                                   │ │
│  │  │    │    ├── SSH:22 (my IP)│      │  ├── deploy-nexus.yaml                       │ │
│  │  │    │    └── TCP:8080 (all)│      │  ├── deploy-node.yaml                        │ │
│  │  │    └── EC2 (Amazon Linux) │      │  ├── deploy-docker-ec2-user.yaml             │ │
│  │  └── entry-script.sh        │      │  ├── deploy-docker-new-user.yaml             │ │
│  │       (user_data bootstrap)  │      │  ├── deploy-docker-with-roles.yaml           │ │
│  │       ├── yum install docker │      │  └── deploy-to-K8s.yaml                     │ │
│  │       ├── systemctl start    │      │                                              │ │
│  │       └── usermod docker grp │      │ Roles                                       │ │
│  │                              │      │  ├── create_user/                            │ │
│  │ Terraform/ (EKS)             │      │  └── state_containers/                      │ │
│  │  ├── providers.tf            │      │      └── files/docker-compose.yaml          │ │
│  │  │    └─ AWS v5.20.1         │      │                                              │ │
│  │  ├── vpc.tf                  │      │ project-vars                                 │ │
│  │  │    └─ module vpc v5.1.2   │      │  (version, linux_name, user_groups,          │ │
│  │  └── eks-cluster.tf          │      │   docker_password)                           │ │
│  │       └─ module eks v19.17.2 │      └──────────────────────────────────────────────┘ │
│  └──────────────┬───────────────┘                         │                             │
└─────────────────┼───────────────────────────────────────  │ ─────────────────────────── ┘
                  │ terraform apply                          │ SSH (port 22) / kubectl
                  │                                         │
                  ▼                           ├──────────────────────────┬─────────────────────────┐
┌─────────────────────────────────────────────┐  │                          │                         │
│        AWS CLOUD  (eu-central-1)            │  │                          │                         │
│                                             │  ▼                          ▼                         ▼
│  ┌──────────────────────────────────────┐   │  ┌──────────────────┐  ┌─────────────────────────────┐  ┌──────────────────────────┐
│  │          myapp-vpc  (VPC)            │   │  │  DIGITALOCEAN    │  │  AWS CLOUD  (ca-central-1)  │  │   AWS EKS CLUSTER        │
│  │  ┌────────────┐  ┌────────────────┐  │   │  │  (static inv.)   │  │  Provisioned by             │  │   myapp-eks-cluster      │
│  │  │  Public    │  │  Private       │  │   │  │──────────────────│  │  terraform-learn/            │  │  Provisioned by          │
│  │  │  Subnets   │  │  Subnets       │  │   │  │ nexus_server     │  │─────────────────────────────│  │  Terraform/ (EKS)        │
│  │  │  (ELB)     │  │  (internal-ELB)│  │   │  │ Ubuntu / root    │  │  docker_server              │  │──────────────────────────│
│  │  └────────────┘  └───────┬────────┘  │   │  │ 165.245.238.35   │  │  Amazon Linux 2 / ec2-user  │  │  K8s version: 1.27       │
│  │         ▲                │           │   │  │──────────────────│  │  3.96.159.160               │  │  Node group: dev         │
│  │         │                ▼           │   │  │ Nexus 3          │  │─────────────────────────────│  │  ├── 3x t2.small nodes   │
│  │  ┌──────────────────────────────┐    │   │  │ /opt/nexus       │  │  Docker pre-installed via   │  │                          │
│  │  │      NAT Gateway             │    │   │  │ OS user: nexus   │  │  entry-script.sh (user_data)│  │  Namespace: my-app       │
│  │  └──────────────────────────────┘    │   │  │ (least priv.)    │  │                             │  │  ┌────────────────────┐  │
│  │                                      │   │  │                  │  │  Ansible adds:              │  │  │ nginx Deployment   │  │
│  │  ┌──────────────────────────────┐    │   │  │ Node.js server   │  │  ├─ docker-compose v2       │  │  │ (nginx-config.yaml │  │
│  │  │   EKS Managed Node Group     │    │   │  │ Ubuntu / root    │  │  ├─ OS user: ndu            │  │  │  applied by        │  │
│  │  │   3x t2.small  (in private   │◄───┘   │  │ 143.198.39.191   │  │  ├─ java-app  :8080        │  │  │  Ansible)          │  │
│  │  │   subnets, tagged for K8s)   │        │  │ OS user: ndu     │  │  ├─ mysql     :3306        │  │  └────────────────────┘  │
│  │  └──────────────────────────────┘        │  └──────────────────┘  │  └─ phpmyadmin:8083        │  │                          │
│  └──────────────────────────────────────────┘                         └─────────────────────────────┘  │  Managed via:            │
│                                                                                                         │  kubernetes.core.k8s     │
│                                                                                                         └──────────────────────────┘
                                                          │  pulls images
                                                          ▼
                                              ┌───────────────────────────┐
                                              │    CONTAINER REGISTRIES   │
                                              │  ┌─────────────────────┐  │
                                              │  │  Docker Hub         │  │
                                              │  │  ndubuisip/         │  │
                                              │  │  demo-app:          │  │
                                              │  │  java-maven-3.0     │  │
                                              │  └─────────────────────┘  │
                                              │  ┌─────────────────────┐  │
                                              │  │  AWS ECR            │  │
                                              │  │  (configurable via  │  │
                                              │  │   role defaults)    │  │
                                              │  └─────────────────────┘  │
                                              └───────────────────────────┘
```

### Two-Stage Automation Pipeline

```
  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │  TRACK A — EC2 Docker Server  (terraform-learn/ → Ansible docker playbooks)      │
  └──────────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
  terraform apply  (terraform-learn/)
         │
         └─── Provisions EC2 instance in ca-central-1
                   ├── VPC + public subnet + Internet Gateway
                   ├── Security Group: SSH:22 (restricted to my_ip), TCP:8080 (public)
                   ├── Amazon Linux 2 AMI (latest, dynamically resolved)
                   ├── Public IP assigned, SSH key: ec2-server-key (~/.ssh/id_rsa.pub)
                   └── entry-script.sh  (user_data bootstrap on first boot)
                             ├── yum install docker
                             ├── systemctl start docker
                             └── usermod -aG docker ec2-user
                                        │
                                        ▼
                              EC2 READY (Docker installed)  ─────────────────────────┐
                                                                                      │
  ┌──────────────────────────────────────────────────────────────────────────────────┐│
  │  Ansible takes over — deploy-docker-*.yaml                                       ││
  └──────────────────────────────────────────────────────────────────────────────────┘│
         │                                                                            │
         ▼   targets: docker_server (static) or tag_Name_prod_server (dynamic) ◄─────┘
         ├─── Install docker-compose v2
         ├─── Create OS user ndu, add to docker group
         └─── Login to registry + start Docker Compose stack (java-app, mysql, phpmyadmin)


  ┌──────────────────────────────────────────────────────────────────────────────────┐
  │  TRACK B — EKS Cluster  (Terraform/ EKS → Ansible K8s playbook)                 │
  └──────────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
  terraform apply  (Terraform/)
         │
         ├─── Provisions VPC (myapp-vpc, 10.0.0.0/16, eu-central-1)
         │         ├── Public subnets   → tagged for AWS Load Balancer
         │         ├── Private subnets  → tagged for internal ELB
         │         └── Single NAT Gateway for outbound access from private subnets
         │
         └─── Provisions EKS Cluster (myapp-eks-cluster, K8s 1.27)
                   ├── Placed in private subnets (no direct internet exposure)
                   ├── Public API endpoint enabled (for kubectl from control node)
                   └── Managed node group: 3x t2.small  (auto-scaling: 1–3 nodes)
                              │
                              ▼
                        EKS CLUSTER READY  ──────────────────────────────────┐
                                                                              │
  ┌─────────────────────────────────────────────────────────────────────────┐│
  │  Ansible takes over — deploy-to-K8s.yaml                                ││
  └─────────────────────────────────────────────────────────────────────────┘│
         │                                                                    │
         ▼   target: localhost  (kubectl context from terraform apply) ◄──────┘
         ├─── Play 1: Create Kubernetes namespace  →  my-app
         └─── Play 2: Apply nginx manifest  →  nginx Deployment in my-app namespace


  ┌─────────────────────────────────────────────────────────────────────────┐
  │  PARALLEL — Multi-Cloud Workload Deployments  (Ansible)                 │
  └─────────────────────────────────────────────────────────────────────────┘
         │
         ▼
  ansible-playbook <playbook>.yaml
         │
         ├─── 1. Resolve Inventory
         │         ├── Static:   hosts file  ─────────────► nexus_server  (DigitalOcean)
         │         │                                         docker_server (AWS EC2)
         │         └── Dynamic:  aws_ec2 plugin ──────────► tag_Name_prod_server, aws_ec2
         │
         ├─── 2. Load Variables
         │         ├── project-vars        (shared vars file)
         │         ├── role defaults/      (lowest precedence)
         │         └── role vars/          (overrides defaults)
         │
         ├─── 3. Execute Plays in Order
         │         ├── Play 1: Install packages / runtime
         │         ├── Play 2: Create OS user  (role or inline)
         │         ├── Play 3: Install Docker Compose
         │         └── Play 4: Deploy & start application containers
         │
         └─── 4. Verify Deployment
                   ├── ps aux | grep <process>
                   └── netstat -plnt  (port listening check)
```

---

## Technologies Used

| Category | Tools |
|---|---|
| Infrastructure Provisioning | Terraform (v5.20.1 AWS provider) |
| Configuration Management | Ansible |
| Cloud Providers | AWS (EC2, EKS, ECR, VPC — `eu-central-1` / `ca-central-1`), DigitalOcean (Droplets) |
| Containerization | Docker, Docker Compose v2 |
| Container Orchestration | AWS EKS (Kubernetes 1.27), `kubernetes.core` Ansible collection |
| Application Runtime | Java (Spring Boot), Node.js, MySQL, phpMyAdmin |
| Artifact Repository | Sonatype Nexus 3 |
| Dynamic Inventory | AWS EC2 Ansible plugin |
| Terraform Modules | `terraform-aws-modules/eks/aws` v19.17.2, `terraform-aws-modules/vpc/aws` v5.1.2 |
| OS | Amazon Linux (AWS EC2), Ubuntu (DigitalOcean Droplets) |
| SSH Auth | RSA key-pair (`~/.ssh/id_rsa`) |

---

## Infrastructure & Inventory

### Static Inventory (`hosts`)

Two server groups defined for targeted playbook execution:

```
[nexus_server]   — DigitalOcean Droplet (Ubuntu), root user — hosts Nexus Repository Manager
[docker_server]  — AWS EC2 (Amazon Linux), ec2-user      — hosts Docker workloads
```

> `deploy-node.yaml` targets a second **DigitalOcean Droplet** (`143.198.39.191`) via a hardcoded IP rather than a named inventory group. Both DigitalOcean servers use Ubuntu and are accessed as `root`, which is the DigitalOcean Droplet default.

### Dynamic Inventory (`inventory_aws_ec2.yaml`)

Uses the **AWS EC2 dynamic inventory plugin** to automatically discover instances in the `ca-central-1` region. Instances are grouped by:
- **EC2 tags** (e.g., `tag_Name_prod_server`) — enabling tag-driven targeting
- **Instance type** — enabling topology-aware playbook targeting

This eliminates the need to maintain static IP lists as infrastructure scales.

### Ansible Configuration (`ansible.cfg`)

- Host key checking disabled for automated pipelines
- Default remote user: `ec2-user`
- SSH private key: `~/.ssh/id_rsa`
- AWS EC2 inventory plugin enabled globally

---

## Project Deliverables

### 1. Nexus Repository Manager Deployment

**Playbook:** `deploy-nexus.yaml`
**Target:** `nexus_server` — DigitalOcean Droplet (Ubuntu, root)

Fully automates the installation and configuration of **Sonatype Nexus 3** on a bare DigitalOcean Ubuntu Droplet — a task typically requiring 15+ manual steps.

**What it does:**
- Updates APT package cache and installs **OpenJDK 8** and **net-tools**
- Downloads the latest Nexus 3 tarball directly from Sonatype
- Unpacks and renames the installation directory to `/opt/nexus`
- Creates a dedicated `nexus` OS group and user (security best practice — no root for services)
- Assigns recursive ownership of `/opt/nexus` and `/opt/sonatype-work` to the `nexus` user
- Configures `nexus.rc` to enforce the `nexus` service account at startup
- Starts the Nexus service
- Verifies the running process via `ps aux` and confirms the listening port via `netstat`

**DevOps value:** Demonstrates idempotent server provisioning, principle of least privilege for service accounts, and automated post-deployment verification.

> `nexus.sh` — The original manual shell script this playbook was derived from, included to illustrate the automation journey from imperative scripting to declarative IaC.

---

### 2. Node.js Application Deployment

**Playbook:** `deploy-node.yaml`
**Target:** DigitalOcean Droplet (`143.198.39.191`, Ubuntu) — hardcoded IP

End-to-end automation of a Node.js application deployment from a packaged artifact onto a DigitalOcean Ubuntu Droplet.

**What it does:**
- Updates APT cache and installs **Node.js** and **npm**
- Creates a dedicated Linux user for the application (drawn from `project-vars`)
- Unpacks a versioned `.tgz` application artifact to the user's home directory
- Installs npm dependencies via the Ansible `npm` module
- Launches the Node.js server asynchronously (non-blocking, `async: 1000 / poll: 0`)
- Confirms the process is running using `ps aux`

**DevOps value:** Demonstrates artifact-based deployments, non-root application users, async task execution, and live process verification.

---

### 3. Docker & Docker Compose Automation

> **Pre-requisite:** The target EC2 instance is provisioned by **`terraform-learn/`** — a Terraform project that creates a VPC, public subnet, security group (SSH:22, TCP:8080), and an Amazon Linux 2 EC2 instance in `ca-central-1`. The `entry-script.sh` user_data script bootstraps the instance on first boot by installing Docker, starting the daemon, and adding `ec2-user` to the `docker` group. Ansible picks up from this point to deploy the application stack.

Three playbooks demonstrate a deliberate learning progression in Docker automation:

#### Stage 1 — EC2 User Deployment (`deploy-docker-ec2-user.yaml`)
**Target:** Static `docker_server` group (Terraform-provisioned EC2, `3.96.159.160`)

- Detects remote CPU architecture dynamically (`uname -m`) for cross-platform binary selection
- Installs the correct Docker Compose v2 binary to `~/.docker/cli-plugins/`
- Adds `ec2-user` to the `docker` group and resets the SSH connection to apply group changes without logout
- Authenticates to Docker Hub and launches a multi-container stack via `docker_compose_v2`

#### Stage 2 — New User Deployment (`deploy-docker-new-user.yaml`)
**Target:** Dynamically tagged EC2 instances (`tag_Name_prod_server`)

- Installs Docker via `yum` and starts the daemon with `systemd`
- Creates a new non-root Linux user (`ndu`) with `admin` and `docker` group membership
- Installs Docker Compose and runs the full application stack as the new user

#### Stage 3 — Role-Based Deployment (`deploy-docker-with-roles.yaml`) *(most advanced)*
**Target:** All AWS EC2 instances via dynamic inventory

- Refactors all inline tasks into reusable **Ansible roles**
- Uses `aws_ec2` dynamic inventory — no hardcoded IPs
- Delegates user creation to the `create_user` role and container management to the `state_containers` role

**Multi-container application stack (via `docker-compose.yaml`):**

| Service | Image | Port | Notes |
|---|---|---|---|
| `java-app` | `ndubuisip/demo-app:java-maven-3.0` | 8080 | Waits for MySQL healthcheck |
| `mysql` | `mysql` | 3306 | Persistent volume, healthcheck via `mysqladmin ping` |
| `phpmyadmin` | `phpmyadmin` | 8083 | Connected to MySQL service |

**DevOps value:** Demonstrates Docker lifecycle management, dynamic inventory targeting, multi-container orchestration, and the shift to reusable role-based IaC — a production-ready pattern.

---

### 4. Role-Based Automation

**Location:** `roles/`

Encapsulates reusable automation logic following the **Ansible Roles standard directory structure**.

#### `create_user` Role

| Component | Content |
|---|---|
| `defaults/main.yaml` | Default value: `user_groups: admin,docker` |
| `tasks/main.yaml` | Creates Linux user `ndu` with configurable group membership |

Designed to be overridden at the playbook level — the consuming playbook can pass different `user_groups` without modifying the role.

#### `state_containers` Role

| Component | Content |
|---|---|
| `defaults/main.yaml` | AWS ECR registry endpoint (placeholder) |
| `vars/main.yaml` | Docker Hub registry URL and credentials |
| `tasks/main.yaml` | Copies compose file, authenticates to registry, starts containers |
| `files/docker-compose.yaml` | Full multi-container stack definition |

Supports **both Docker Hub and AWS ECR** as target registries — the registry endpoint is configurable via variable precedence (`vars` overrides `defaults`).

**DevOps value:** Demonstrates separation of concerns, DRY (Don't Repeat Yourself) infrastructure code, and Ansible's variable precedence model.

---

### 5. EKS Cluster Provisioning + Kubernetes Application Deployment

This deliverable is a **two-stage pipeline** combining Terraform and Ansible — the EKS cluster must exist before Ansible can deploy to it.

---

#### Stage 1 — EKS Infrastructure Provisioning (Terraform)

**Directory:** `../Terraform/`
**Region:** `eu-central-1`

Provisions a production-ready EKS environment from scratch using community-maintained Terraform modules.

**`vpc.tf` — Networking Layer**
- Creates `myapp-vpc` with CIDR `10.0.0.0/16`
- Provisions **public subnets** (tagged `kubernetes.io/role/elb` for AWS Load Balancer Controller)
- Provisions **private subnets** (tagged `kubernetes.io/role/internal-elb` for internal load balancers)
- Deploys a **single NAT Gateway** to allow outbound internet access from private subnets
- Enables DNS hostnames for service discovery within the VPC
- Dynamically queries available AZs via `data.aws_availability_zones`

**`eks-cluster.tf` — Cluster Layer**
- Deploys `myapp-eks-cluster` running **Kubernetes 1.27** using `terraform-aws-modules/eks/aws` v19.17.2
- EKS API endpoint is **publicly accessible** (for kubectl from the control node)
- Worker nodes placed in **private subnets** (security best practice — no direct internet exposure)
- Managed node group `dev`:

| Setting | Value |
|---|---|
| Instance type | `t2.small` |
| Desired nodes | 3 |
| Minimum nodes | 1 |
| Maximum nodes | 3 |

- Cluster tagged `environment=development` and `application=myapp`

**DevOps value:** Demonstrates Terraform module composition, VPC design for Kubernetes (subnet tagging for ELB integration), EKS managed node groups, and infrastructure-as-code for cloud-native platforms.

---

#### Stage 2 — Application Deployment (Ansible)

**Playbook:** `deploy-to-K8s.yaml`
**Target:** `localhost` (uses the kubectl context configured after `terraform apply`)

Once the EKS cluster is running and `kubectl` is configured to point at it, Ansible takes over for application-layer deployment using the `kubernetes.core` collection.

**What it does:**
- Creates a dedicated Kubernetes namespace (`my-app`) declaratively
- Deploys an nginx application from a local Kubernetes manifest (`nginx-config.yaml`) into the `my-app` namespace

**DevOps value:** Demonstrates the integration of two IaC tools in a single pipeline — Terraform owns infrastructure, Ansible owns application state. This is a common pattern in enterprise DevOps workflows where platform and application teams use different toolchains on the same cluster.

---

## Ansible Roles

```
roles/
├── create_user/
│   ├── defaults/main.yaml      # Default group membership
│   └── tasks/main.yaml         # User creation task
│
└── state_containers/
    ├── defaults/main.yaml       # AWS ECR registry (overridable)
    ├── vars/main.yaml           # Docker Hub registry credentials
    ├── tasks/main.yaml          # Login, copy, and start containers
    └── files/
        └── docker-compose.yaml  # Java app + MySQL + phpMyAdmin stack
```

---

## Key Outcomes & Skills Demonstrated

| Skill | Evidence |
|---|---|
| **Terraform Infrastructure Provisioning** | Two separate Terraform projects — EC2 instance for Docker (`terraform-learn/`) and full EKS cluster (`Terraform/`) |
| **Terraform Module Composition** | EKS VPC module composed with EKS module via output references; EC2 project uses raw resource blocks |
| **Kubernetes Networking Design** | Subnet tagging for ELB and internal-ELB, NAT Gateway for private node egress |
| **Multi-Tool IaC Pipeline** | Terraform provisions EC2 + EKS infrastructure → Ansible deploys applications on both targets |
| **EC2 Bootstrap via user_data** | `entry-script.sh` pre-installs Docker at instance launch; Ansible handles app-layer config on top |
| **Ansible Playbook Authorship** | 6 production-style playbooks covering diverse deployment scenarios |
| **Role-Based IaC Design** | 2 reusable roles with proper `defaults`, `vars`, `tasks`, and `files` separation |
| **Dynamic Cloud Inventory** | AWS EC2 plugin with tag-based and instance-type-based grouping |
| **Multi-Stage Deployment Pipelines** | Sequential plays within a single playbook (install → configure → deploy → verify) |
| **Multi-Cloud Deployment** | Targets DigitalOcean Droplets (Nexus, Node.js) and AWS EC2 (Docker) from a single Ansible control node |
| **Security Best Practices** | Private EKS nodes, dedicated service accounts, non-root app users, credential parameterization |
| **Docker Automation** | Docker install, group management, Compose v2, registry authentication |
| **Container Orchestration** | EKS managed node groups (Terraform) + K8s namespace/workload management (Ansible) |
| **Cross-Platform Awareness** | Dynamic architecture detection for binary downloads |
| **Artifact-Based Deployment** | Versioned application packaging and unpack-and-run deployment |
| **Post-Deployment Verification** | Process checks, port validation, and debug output built into playbooks |
| **IaC Progression** | Shell script → inline playbook → parameterized playbook → role-based playbook → Terraform + Ansible |

---

## Project Structure

```
DevOps-Project/
│
├── Terraform/
│   └── terraform-learn/                 # Track A — EC2 Docker server provisioning (ca-central-1)
│       ├── providers.tf                 # AWS provider ~> 6.0
│       ├── main.tf                      # VPC, subnet, IGW, security group, EC2 instance + SSH key
│       ├── entry-script.sh              # user_data: installs Docker, starts daemon, adds ec2-user to group
│       └── terraform.tfvars            # Variable values (CIDR, AZ, instance type, SSH key path)
│
└── Configuration-Management-with-Ansible/
    │
    ├── Terraform/                       # Track B — EKS cluster provisioning (eu-central-1)
    │   ├── providers.tf                 # AWS provider v5.20.1
    │   ├── vpc.tf                       # VPC, public/private subnets, NAT Gateway
    │   ├── eks-cluster.tf               # EKS cluster (K8s 1.27) + managed node group
    │   └── .terraform.lock.hcl         # Provider version lock file
│
    └── ansible-projects/                # Configuration management & application deployment
    ├── ansible.cfg                      # Ansible global configuration
    ├── hosts                            # Static inventory (nexus, docker servers)
    ├── inventory_aws_ec2.yaml           # Dynamic AWS EC2 inventory (ca-central-1)
    ├── project-vars                     # Shared variable file
    │
    ├── my-playbook.yaml                 # Basic nginx playbook (initial exercise)
    ├── deploy-node.yaml                 # Node.js application deployment
    ├── deploy-nexus.yaml                # Sonatype Nexus 3 full installation
    ├── deploy-docker-ec2-user.yaml      # Docker deployment — ec2-user (static inventory)
    ├── deploy-docker-new-user.yaml      # Docker deployment — new Linux user (dynamic inventory)
    ├── deploy-docker-with-roles.yaml    # Docker deployment — role-based (dynamic inventory)
    ├── deploy-to-K8s.yaml               # Kubernetes namespace + nginx deployment (post-EKS)
    │
    ├── nexus.sh                         # Manual Nexus install script (pre-automation reference)
    │
    └── roles/
        ├── create_user/
        │   ├── defaults/main.yaml       # Default: user_groups = admin,docker
        │   └── tasks/main.yaml          # Creates Linux user with group membership
        └── state_containers/
            ├── defaults/main.yaml       # AWS ECR registry (overridable)
            ├── vars/main.yaml           # Docker Hub registry URL + credentials
            ├── tasks/main.yaml          # Copy compose, login to registry, start containers
            └── files/
                └── docker-compose.yaml  # Java app + MySQL + phpMyAdmin stack
```

---

*Built as part of a hands-on DevOps engineering curriculum covering infrastructure provisioning with Terraform, configuration management with Ansible, cloud-native Kubernetes on AWS EKS, and container-based application delivery.*
