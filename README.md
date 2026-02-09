# Implementing-IaC-and-CI-CD-for-a-spring-boot-application
A full Documentation


# Schedemy - Infrastructure & Deployment Report

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture Diagram](#2-architecture-diagram)
3. [Infrastructure as Code - Terraform](#3-infrastructure-as-code---terraform)
   - 3.1 [Provider & AMI Configuration (main.tf)](#31-provider--ami-configuration---maintf)
   - 3.2 [Network Foundation - VPC (main.tf)](#32-network-foundation---vpc-maintf)
   - 3.3 [Subnets (main.tf)](#33-subnets-maintf)
   - 3.4 [Routing (main.tf)](#34-routing-maintf)
   - 3.5 [Security Groups (main.tf)](#35-security-groups-maintf)
   - 3.6 [Application Load Balancer (compute.tf)](#36-application-load-balancer---computetf)
   - 3.7 [Launch Template (compute.tf)](#37-launch-template---computetf)
   - 3.8 [Auto Scaling Group (compute.tf)](#38-auto-scaling-group---computetf)
   - 3.9 [RDS MySQL Database (database.tf)](#39-rds-mysql-database---databasetf)
   - 3.10 [Variables & Outputs](#310-variables--outputs)
4. [Configuration Management - Ansible](#4-configuration-management---ansible)
   - 4.1 [Dynamic Inventory (inventory.aws_ec2.yml)](#41-dynamic-inventory---inventoryaws_ec2yml)
   - 4.2 [Ansible Configuration (ansible.cfg)](#42-ansible-configuration---ansiblecfg)
   - 4.3 [Deployment Playbook (deploy.yml)](#43-deployment-playbook---deployyml)
5. [CI/CD Pipeline - GitHub Actions](#5-cicd-pipeline---github-actions)
   - 5.1 [Workflow Overview](#51-workflow-overview)
   - 5.2 [Pipeline Steps Breakdown](#52-pipeline-steps-breakdown)
   - 5.3 [GitHub Secrets Required](#53-github-secrets-required)
6. [Application Configuration - Spring Boot](#6-application-configuration---spring-boot)
   - 6.1 [Default Profile (H2 - Development)](#61-default-profile-h2---development)
   - 6.2 [Production Profile (MySQL - AWS RDS)](#62-production-profile-mysql---aws-rds)
7. [Complete Deployment Workflow](#7-complete-deployment-workflow)
8. [Security Considerations](#8-security-considerations)
9. [Resource Summary](#9-resource-summary)

---

## 1. Project Overview

**Schedemy** is a university course scheduling web application built with **Java Spring Boot 2.7.2**. The backend provides REST APIs for managing courses, instructors, rooms, schedules, and time slots, with an email notification feature for schedule distribution.

| Attribute         | Value                          |
|-------------------|--------------------------------|
| **Application**   | Schedemy - Course Scheduler    |
| **Language**      | Java 17                        |
| **Framework**     | Spring Boot 2.7.2              |
| **Build Tool**    | Apache Maven                   |
| **Database (Dev)**| H2 In-Memory                   |
| **Database (Prod)**| AWS RDS MySQL 8.0             |
| **Cloud Provider**| Amazon Web Services (AWS)      |
| **IaC Tool**      | Terraform                      |
| **Config Mgmt**   | Ansible                        |
| **CI/CD**         | GitHub Actions                 |
| **Region**        | us-east-1 (N. Virginia)        |

---

## 2. Architecture Diagram

```
                        ┌─────────────────────────────────────────────────────┐
                        │                  AWS Cloud (us-east-1)              │
                        │                                                     │
    Internet            │   ┌──────────────────────────────────────────┐      │
       │                │   │          VPC: 10.0.0.0/16                │      │
       │                │   │                                          │      │
       ▼                │   │   ┌──────────────────────────────────┐   │      │
  ┌─────────┐           │   │   │     Public Subnets               │   │      │
  │  Users   │──HTTP──▶│   │   │                                  │   │      │
  └─────────┘   :80     │   │   │  ┌─────────────────────────┐    │   │      │
                        │   │   │  │  Application Load        │    │   │      │
                        │   │   │  │  Balancer (ALB)          │    │   │      │
                        │   │   │  │  Port 80 → Port 8080     │    │   │      │
                        │   │   │  └────────┬────────────┬────┘    │   │      │
                        │   │   │           │            │         │   │      │
                        │   │   │     ┌─────▼──────┐ ┌───▼──────┐ │   │      │
                        │   │   │     │ EC2 App    │ │ EC2 App  │ │   │      │
                        │   │   │     │ Server 1   │ │ Server 2 │ │   │      │
                        │   │   │     │ (AZ: 1a)   │ │ (AZ: 1b) │ │   │      │
                        │   │   │     │ Port 8080  │ │ Port 8080│ │   │      │
                        │   │   │     └─────┬──────┘ └───┬──────┘ │   │      │
                        │   │   │           │            │         │   │      │
                        │   │   │     Auto Scaling Group (2-4)     │   │      │
                        │   │   └──────────────────────────────────┘   │      │
                        │   │                                          │      │
                        │   │   ┌──────────────────────────────────┐   │      │
                        │   │   │     Private Subnets              │   │      │
                        │   │   │                                  │   │      │
                        │   │   │     ┌─────────────────────┐      │   │      │
                        │   │   │     │  RDS MySQL 8.0      │      │   │      │
                        │   │   │     │  db.t3.micro        │      │   │      │
                        │   │   │     │  Port 3306          │      │   │      │
                        │   │   │     │  (Private - No      │      │   │      │
                        │   │   │     │   Public Access)    │      │   │      │
                        │   │   │     └─────────────────────┘      │   │      │
                        │   │   └──────────────────────────────────┘   │      │
                        │   └──────────────────────────────────────────┘      │
                        └─────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────┐
  │  CI/CD Pipeline (GitHub Actions)                              │
  │                                                                │
  │  Push to main ──▶ Checkout ──▶ Install Ansible ──▶ SSH to    │
  │                    Code        + boto3 + aws       EC2        │
  │                                collection        instances    │
  │                                                    │          │
  │                                              ▼          │
  │                                         Ansible Playbook      │
  │                                         (git clone, mvn       │
  │                                          build, systemd       │
  │                                          service start)       │
  └──────────────────────────────────────────────────────────────┘
```

---

## 3. Infrastructure as Code - Terraform

Terraform is an **Infrastructure as Code (IaC)** tool that allows us to define, provision, and manage cloud resources using declarative configuration files. Instead of manually clicking through the AWS Console, we write `.tf` files that describe what we want, and Terraform creates it.

**Total Resources Provisioned: 19**

### 3.1 Provider & AMI Configuration - `main.tf`

```hcl
terraform {
  required_version = ">= 1.3.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

**What this does:**
- Declares that we are using the **AWS provider** from HashiCorp's registry.
- Sets the deployment region to **us-east-1 (N. Virginia)** — this is where all our infrastructure will live.
- Requires Terraform version 1.3.0 or newer to ensure compatibility.

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical (Ubuntu publisher)
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}
```

**What this does:**
- Dynamically fetches the **latest Ubuntu 22.04 LTS AMI** from Canonical instead of hardcoding an AMI ID.
- **Why this matters:** AMI IDs change frequently as Canonical publishes updates. A hardcoded ID could become deregistered and break the deployment.

---

### 3.2 Network Foundation - VPC (`main.tf`)

```hcl
resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
}
```

**What this provides:**
- A **Virtual Private Cloud (VPC)** — our own isolated network within AWS.
- The CIDR block `10.0.0.0/16` provides **65,536 IP addresses** for all our resources.
- DNS support is enabled so instances can resolve domain names (required for the app to reach RDS by hostname).

```hcl
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main_vpc.id
}
```

**What this provides:**
- An **Internet Gateway** attached to the VPC — this is the "door" that allows our public subnets to communicate with the internet.
- Without this, no resource in the VPC could reach or be reached from the internet.

---

### 3.3 Subnets (`main.tf`)

We create **4 subnets** across **2 Availability Zones** for high availability:

| Subnet | CIDR | AZ | Type | Purpose |
|--------|------|----|------|---------|
| public-subnet-1 | 10.0.1.0/24 | us-east-1a | Public | ALB + App Server 1 |
| public-subnet-2 | 10.0.2.0/24 | us-east-1b | Public | ALB + App Server 2 |
| private-subnet-1 | 10.0.3.0/24 | us-east-1a | Private | RDS Database |
| private-subnet-2 | 10.0.4.0/24 | us-east-1b | Private | RDS Database (Multi-AZ requirement) |

**What this provides:**
- **Public subnets** have `map_public_ip_on_launch = true`, meaning instances get public IPs automatically. This is needed for the ALB to serve traffic and for Ansible to SSH into instances.
- **Private subnets** have no public IP assignment. The database lives here and is **completely isolated from the internet** — only app servers can reach it.
- **Two AZs** provide fault tolerance: if `us-east-1a` goes down, `us-east-1b` keeps serving traffic.

---

### 3.4 Routing (`main.tf`)

```hcl
resource "aws_route_table" "public_rt" {
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
}
```

**What this provides:**
- A **route table** that directs all outbound traffic (`0.0.0.0/0`) to the Internet Gateway.
- Associated with both public subnets so they can send/receive internet traffic.
- Private subnets use the **default route table** which has no internet route — ensuring database isolation.

---

### 3.5 Security Groups (`main.tf`)

Security groups act as **virtual firewalls** for our resources. We define three:

#### ALB Security Group (`alb_sg`)
| Direction | Port | Protocol | Source | Purpose |
|-----------|------|----------|--------|---------|
| Inbound | 80 | TCP | 0.0.0.0/0 (Anywhere) | Accept HTTP from the internet |
| Outbound | All | All | 0.0.0.0/0 | Allow all outbound traffic |

#### App Server Security Group (`app_sg`)
| Direction | Port | Protocol | Source | Purpose |
|-----------|------|----------|--------|---------|
| Inbound | 8080 | TCP | alb_sg (ALB only) | Only ALB can reach Spring Boot |
| Inbound | 22 | TCP | 0.0.0.0/0 | SSH access for Ansible deployment |
| Outbound | All | All | 0.0.0.0/0 | Allow all outbound traffic |

#### Database Security Group (`db_sg`)  — defined in `database.tf`
| Direction | Port | Protocol | Source | Purpose |
|-----------|------|----------|--------|---------|
| Inbound | 3306 | TCP | app_sg (App Servers only) | Only app servers can access MySQL |
| Outbound | All | All | 0.0.0.0/0 | Allow all outbound traffic |

**What this provides:**
- **Defense in depth**: Traffic flows Internet → ALB (port 80) → App (port 8080) → DB (port 3306). Each layer only accepts traffic from the layer above.
- The database is **double-protected**: it's in a private subnet AND its security group only allows connections from the app server security group.

---

### 3.6 Application Load Balancer - `compute.tf`

```hcl
resource "aws_lb" "schedemy_alb" {
  name               = "schedemy-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_sg.id]
  subnets            = [aws_subnet.public_1.id, aws_subnet.public_2.id]
}
```

**What this provides:**
- An **Application Load Balancer (ALB)** — the single entry point for all user traffic.
- Sits in **both public subnets** across two AZs for high availability.
- Listens on **port 80 (HTTP)** and forwards traffic to the target group on **port 8080**.

```hcl
resource "aws_lb_target_group" "schedemy_tg" {
  port     = 8080
  protocol = "HTTP"
  health_check {
    path                = "/"
    healthy_threshold   = 2
    unhealthy_threshold = 5
    interval            = 30
    matcher             = "200-399"
  }
}
```

**What the health check does:**
- Every **30 seconds**, the ALB sends an HTTP request to `/` on port 8080.
- If the instance responds with a status code between **200-399**, it's considered healthy.
- After **2 consecutive** successful checks → instance is marked **healthy** (receives traffic).
- After **5 consecutive** failed checks → instance is marked **unhealthy** (removed from rotation).

**Why this matters:** If one app server crashes, the ALB automatically stops sending traffic to it while the Auto Scaling Group replaces it.

---

### 3.7 Launch Template - `compute.tf`

```hcl
resource "aws_launch_template" "schedemy_lt" {
  image_id      = data.aws_ami.ubuntu.id
  instance_type = var.instance_type     # t3.micro
  key_name      = var.key_name          # SSH key for Ansible
}
```

**What this provides:**
- A **blueprint** for creating EC2 instances — defines the OS image, instance size, and SSH key.
- Uses `data.aws_ami.ubuntu.id` to always get the latest Ubuntu 22.04 image.
- Instance type `t3.micro` is **Free Tier eligible**.

**User Data (Bootstrap Script):**
```bash
#!/bin/bash
apt-get update -y
apt-get install -y openjdk-17-jdk maven docker.io git acl
systemctl start docker && systemctl enable docker
docker run -d -p 8080:80 nginx   # Placeholder for health checks
```

**What this does:**
- Runs automatically when each instance launches (before Ansible reaches it).
- Pre-installs **Java 17, Maven, Docker, and Git** — reducing Ansible deployment time.
- Starts a temporary **Nginx container on port 8080** so the ALB health checks pass immediately, keeping the instance in the target group while Ansible deploys the real application.

---

### 3.8 Auto Scaling Group - `compute.tf`

```hcl
resource "aws_autoscaling_group" "schedemy_asg" {
  desired_capacity    = 2
  max_size            = 4
  min_size            = 2
  vpc_zone_identifier = [aws_subnet.public_1.id, aws_subnet.public_2.id]
  target_group_arns   = [aws_lb_target_group.schedemy_tg.arn]
}
```

**What this provides:**
- Manages a fleet of EC2 instances that run our application.
- **Desired: 2 instances** — one in each AZ for redundancy.
- **Minimum: 2** — never goes below 2 instances (always available).
- **Maximum: 4** — can scale up to 4 instances under load.
- Instances are automatically registered with the ALB target group.

**Tags propagated to instances:**
- `Name: Schedemy-App-Instance` — for identification in the AWS Console.
- `Role: SchedemyWebServer` — **critical** for Ansible's dynamic inventory to discover these instances.

---

### 3.9 RDS MySQL Database - `database.tf`

```hcl
resource "aws_db_instance" "schedemy_db" {
  identifier       = "schedemy-prod-db"
  engine           = "mysql"
  engine_version   = "8.0"
  instance_class   = "db.t3.micro"
  allocated_storage = 20
  db_name          = "schedemy"
  username         = var.db_username
  password         = var.db_password
  publicly_accessible = false
}
```

| Attribute | Value | Explanation |
|-----------|-------|-------------|
| Engine | MySQL 8.0 | Industry standard relational database |
| Instance Class | db.t3.micro | Free Tier eligible, suitable for development |
| Storage | 20 GB (gp2) | General purpose SSD, Free Tier eligible |
| Multi-AZ | false | Single AZ (set to true for production HA) |
| Public Access | false | **Security best practice** — not accessible from internet |
| Subnet Group | Private subnets only | Database lives in isolated private subnets |

**What this provides:**
- A fully managed **MySQL 8.0 database** with automated backups, patching, and maintenance handled by AWS.
- Credentials are stored in Terraform variables (sensitive) — not hardcoded.
- Only accessible from the app server security group on port 3306.

---

### 3.10 Variables & Outputs

#### Variables (`variables.tf`)

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `db_username` | string | "admin" | RDS master username |
| `db_password` | string (sensitive) | — | RDS master password |
| `key_name` | string | — | EC2 SSH key pair name |
| `instance_type` | string | "t3.micro" | EC2 instance size |
| `db_instance_class` | string | "db.t3.micro" | RDS instance size |

The `db_password` is marked as **sensitive** so Terraform never displays it in logs or plan output.

#### Outputs (`outputs.tf`)

| Output | Description | Used By |
|--------|-------------|---------|
| `alb_dns_name` | ALB public URL | End users access the app here |
| `db_endpoint` | RDS host:port | Application database connection |
| `db_address` | RDS hostname only | Ansible injects this into the app |
| `db_name` | Database name | Ansible injects this into the app |
| `vpc_id` | VPC identifier | Reference for debugging |

---

## 4. Configuration Management - Ansible

Ansible is an **agentless configuration management tool** that connects to servers via SSH and configures them. Unlike Terraform (which creates infrastructure), Ansible configures the **software inside** the infrastructure.

### 4.1 Dynamic Inventory - `inventory.aws_ec2.yml`

```yaml
plugin: aws_ec2
regions:
  - us-east-1
filters:
  "tag:Role": SchedemyWebServer
  instance-state-name: running
keyed_groups:
  - key: tags.Role
    prefix: role
compose:
  ansible_host: public_ip_address
```

**What this does:**
- Uses the `aws_ec2` plugin to **dynamically discover** EC2 instances instead of hardcoding IP addresses.
- Filters for instances tagged `Role: SchedemyWebServer` that are in **running** state.
- Groups them under `role_SchedemyWebServer` which the playbook targets.
- Uses each instance's **public IP** for SSH connections.

**Why dynamic inventory:** With Auto Scaling, instances are created and destroyed automatically. Their IP addresses change. Hardcoded IPs would break — dynamic inventory always finds the current instances.

### 4.2 Ansible Configuration - `ansible.cfg`

```ini
[defaults]
inventory = inventory.aws_ec2.yml
host_key_checking = False
remote_user = ubuntu
private_key_file = ~/.ssh/id_rsa
```

| Setting | Value | Purpose |
|---------|-------|---------|
| `inventory` | inventory.aws_ec2.yml | Points to dynamic inventory |
| `host_key_checking` | False | Skip SSH fingerprint verification (required for CI/CD) |
| `remote_user` | ubuntu | Default user on Ubuntu EC2 instances |
| `private_key_file` | ~/.ssh/id_rsa | SSH key for authentication |

### 4.3 Deployment Playbook - `deploy.yml`

The playbook runs **7 sequential tasks** on every discovered EC2 instance:

#### Task 1: Install Dependencies
```yaml
- name: Update Cache and Install Java, Maven, Docker
  apt:
    name: [openjdk-17-jdk, maven, docker.io, git, acl]
    state: present
    update_cache: yes
```
Ensures all required software is installed. Idempotent — won't reinstall if already present.

#### Task 2: Clean Project Directory
```yaml
- name: Remove old project directory if exists
  file:
    path: /opt/schedemy
    state: absent
```
Removes any previous deployment to ensure a clean build.

#### Task 3: Clone Code from GitHub
```yaml
- name: Clone project from GitHub
  git:
    repo: https://github.com/Serfuh-beeb/HTU-Schedemy-Website-Backend.git
    dest: /opt/schedemy
    version: main
    force: yes
```
Clones the **latest main branch** directly from GitHub onto each EC2 instance.

#### Task 4: Stop Placeholder Nginx
```yaml
- name: Stop placeholder Nginx container
  shell: docker stop $(docker ps -q) || true
```
Stops the temporary Nginx container that was run by the launch template's user data script.

#### Task 5: Build with Maven
```yaml
- name: Build Application with Maven
  command: mvn clean package -DskipTests
  args:
    chdir: /opt/schedemy
```
Compiles the Java source code and packages it into a runnable JAR file: `EduSched-0.0.1-SNAPSHOT.jar`.

#### Task 6: Create Systemd Service
```yaml
- name: Create Systemd Service for Schedemy
  copy:
    dest: /etc/systemd/system/schedemy.service
    content: |
      [Service]
      Environment="SPRING_PROFILES_ACTIVE=prod"
      Environment="DB_HOST={{ db_host }}"
      Environment="DB_NAME={{ db_name }}"
      Environment="DB_USERNAME={{ db_username }}"
      Environment="DB_PASSWORD={{ db_password }}"
      ExecStart=/usr/bin/java -jar /opt/schedemy/target/EduSched-0.0.1-SNAPSHOT.jar
      Restart=always
```
**What this does:**
- Creates a **systemd service** so the app runs as a managed Linux service.
- Sets `SPRING_PROFILES_ACTIVE=prod` to activate the production database configuration.
- Injects **database connection details** as environment variables.
- `Restart=always` ensures the app automatically restarts if it crashes.

#### Task 7: Start the Service
```yaml
- name: Reload Systemd and Start App
  systemd:
    name: schedemy
    state: restarted
    enabled: yes
    daemon_reload: yes
```
Reloads systemd to pick up the new service file, starts the app, and enables it to start on boot.

---

## 5. CI/CD Pipeline - GitHub Actions

### 5.1 Workflow Overview

The pipeline is defined in `.github/workflows/deploy.yml` and triggers **automatically on every push to the `main` branch**.

```
Developer pushes code → GitHub Actions triggers → Ansible deploys to EC2
```

### 5.2 Pipeline Steps Breakdown

| Step | Action | Purpose |
|------|--------|---------|
| 1. Checkout Code | `actions/checkout@v4` | Downloads the repository code onto the runner |
| 2. Set up Python | `actions/setup-python@v4` | Installs Python 3.x (Ansible dependency) |
| 3. Install Ansible & AWS Tools | `pip install ansible boto3 botocore` + `ansible-galaxy collection install amazon.aws` | Installs Ansible, AWS SDK, and the AWS EC2 inventory plugin |
| 4. Configure AWS Credentials | `aws-actions/configure-aws-credentials@v4` | Sets up AWS access keys so Ansible can discover EC2 instances |
| 5. Setup SSH Key | Write `EC2_SSH_KEY` to `~/.ssh/id_rsa` | Enables SSH access to EC2 instances for Ansible |
| 6. Verify Hosts | `ansible-inventory --list` | Confirms Ansible can discover the EC2 instances before deploying |
| 7. Run Ansible Playbook | `ansible-playbook deploy.yml -vvv` | Executes the full deployment on all discovered instances |

### 5.3 GitHub Secrets Required

These secrets must be configured in **GitHub → Repository → Settings → Secrets and Variables → Actions**:

| Secret Name | Description | Example |
|-------------|-------------|---------|
| `AWS_ACCESS_KEY_ID` | IAM user access key | `AKIA...` |
| `AWS_SECRET_ACCESS_KEY` | IAM user secret key | `wJal...` |
| `EC2_SSH_KEY` | Full contents of the `.pem` file | `-----BEGIN RSA PRIVATE KEY-----...` |
| `DB_HOST` | RDS endpoint hostname | `schedemy-prod-db.xxxxx.us-east-1.rds.amazonaws.com` |
| `DB_NAME` | Database name | `schedemy` |
| `DB_USERNAME` | Database username | `admin` |
| `DB_PASSWORD` | Database password | `SuperSecretPass123!` |

---

## 6. Application Configuration - Spring Boot

The application uses **Spring Profiles** to switch between development and production configurations.

### 6.1 Default Profile (H2 - Development)

File: `src/main/resources/application.properties`

```properties
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driverClassName=org.h2.Driver
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.h2.console.enabled=true
server.port=8080
```

- Uses **H2 in-memory database** for local development — no external database needed.
- H2 Console available at `/h2` for debugging.
- Data is lost on restart (by design for development).

### 6.2 Production Profile (MySQL - AWS RDS)

File: `src/main/resources/application-prod.properties`

```properties
spring.datasource.url=jdbc:mysql://${DB_HOST}:3306/${DB_NAME}
spring.datasource.driverClassName=com.mysql.cj.jdbc.Driver
spring.jpa.database-platform=org.hibernate.dialect.MySQL8Dialect
spring.h2.console.enabled=false
server.port=8080
```

- Activated by setting `SPRING_PROFILES_ACTIVE=prod` (done in the systemd service).
- Connects to **AWS RDS MySQL** using environment variables (`DB_HOST`, `DB_NAME`, `DB_USERNAME`, `DB_PASSWORD`).
- H2 console is **disabled** in production for security.
- `spring.jpa.hibernate.ddl-auto=update` automatically creates/updates database tables based on JPA entities.

**Dependency added to `pom.xml`:**
```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
</dependency>
```

---

## 7. Complete Deployment Workflow

Here is the full end-to-end flow from code change to running application:

### Phase 1: Infrastructure Provisioning (One-time setup)

```bash
cd terraform/
terraform init          # Download AWS provider
terraform apply         # Create all 19 AWS resources
```

This creates the entire infrastructure in approximately 5-10 minutes.

### Phase 2: Continuous Deployment (Every code push)

```
1. Developer pushes code to 'main' branch on GitHub
          │
          ▼
2. GitHub Actions workflow triggers automatically
          │
          ▼
3. Runner installs Ansible + AWS SDK + amazon.aws collection
          │
          ▼
4. AWS credentials are configured from GitHub Secrets
          │
          ▼
5. SSH key is written to runner's filesystem
          │
          ▼
6. Dynamic inventory discovers EC2 instances by Role tag
          │
          ▼
7. Ansible connects via SSH to each EC2 instance and:
   a. Installs Java 17, Maven, Docker, Git
   b. Cleans old deployment
   c. Clones latest code from GitHub
   d. Stops placeholder Nginx container
   e. Builds JAR with Maven (mvn clean package)
   f. Creates systemd service with DB config
   g. Starts the Spring Boot application
          │
          ▼
8. Application is live on ALB DNS endpoint (port 80)
```

### Accessing the Application

```
http://schedemy-alb-291609282.us-east-1.elb.amazonaws.com
```

The ALB receives HTTP traffic on port 80 and distributes it across the healthy EC2 instances running the Spring Boot app on port 8080.

---

## 8. Security Considerations

| Layer | Security Measure | Implementation |
|-------|-----------------|----------------|
| **Network** | VPC Isolation | All resources in a dedicated VPC (10.0.0.0/16) |
| **Network** | Subnet Segmentation | Database in private subnets, app in public subnets |
| **Network** | Security Groups | Layered firewall rules — ALB → App → DB chain |
| **Database** | No Public Access | `publicly_accessible = false` on RDS |
| **Database** | SG Restriction | Only app_sg can reach port 3306 |
| **Secrets** | Terraform Variables | DB password marked as `sensitive` — hidden from logs |
| **Secrets** | GitHub Secrets | AWS keys, SSH key, DB credentials stored encrypted |
| **Secrets** | Environment Variables | DB credentials injected at runtime, not in code |
| **Application** | Spring Profiles | Production config separate from development |
| **Application** | H2 Console Disabled | `spring.h2.console.enabled=false` in production |

---

## 9. Resource Summary

### AWS Resources Created (19 total)

| # | Resource Type | Name | Purpose |
|---|--------------|------|---------|
| 1 | VPC | schedemy-vpc-high-tier | Isolated network |
| 2 | Internet Gateway | schedemy-igw | Internet access for public subnets |
| 3 | Public Subnet 1 | public-subnet-1 (10.0.1.0/24, AZ 1a) | ALB + App Server |
| 4 | Public Subnet 2 | public-subnet-2 (10.0.2.0/24, AZ 1b) | ALB + App Server |
| 5 | Private Subnet 1 | private-subnet-1 (10.0.3.0/24, AZ 1a) | Database |
| 6 | Private Subnet 2 | private-subnet-2 (10.0.4.0/24, AZ 1b) | Database |
| 7 | Route Table | public-route-table | Routes public traffic to IGW |
| 8 | RT Association 1 | — | Links public-subnet-1 to route table |
| 9 | RT Association 2 | — | Links public-subnet-2 to route table |
| 10 | Security Group | schedemy-alb-sg | Firewall for ALB (port 80) |
| 11 | Security Group | schedemy-app-sg | Firewall for App (port 8080, 22) |
| 12 | Security Group | schedemy-db-sg | Firewall for DB (port 3306) |
| 13 | DB Subnet Group | schedemy-db-subnet-group | Defines where RDS can live |
| 14 | RDS Instance | schedemy-prod-db | MySQL 8.0 database |
| 15 | Load Balancer | schedemy-alb | Distributes traffic across instances |
| 16 | LB Listener | — | Listens on port 80, forwards to TG |
| 17 | Target Group | schedemy-tg | Health checks + routes to port 8080 |
| 18 | Launch Template | schedemy-lt-* | EC2 instance blueprint |
| 19 | Auto Scaling Group | schedemy-asg | Manages 2-4 EC2 instances |

### Tools & Technologies Used

| Tool | Version | Purpose |
|------|---------|---------|
| Terraform | >= 1.3.0 | Infrastructure provisioning |
| AWS Provider | ~6.31.0 | Terraform AWS resource management |
| Ansible | Latest | Configuration management & deployment |
| GitHub Actions | v4 | CI/CD automation |
| Java | 17 | Application runtime |
| Spring Boot | 2.7.2 | Web application framework |
| Maven | Latest | Build tool |
| MySQL | 8.0 | Production database |
| Ubuntu | 22.04 LTS | Server operating system |

---

*Report generated for the Schedemy project — HTU Final Project Presentation.*
