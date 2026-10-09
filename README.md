# Expert Listing — Senior DevOps Engineer Challenge Submission

This repository contains the comprehensive technical challenge submission for the Senior DevOps Engineer position at Expert Listing Limited. The solutions focus on introducing robust Infrastructure as Code (IaC), automated CI/CD pipelines, reliable cloud architecture, and clear infrastructure guardrails while keeping implementation pragmatic for a fast-growing proptech platform.

---

## PART A: INFRASTRUCTURE DESIGN & ZERO-DOWNTIME MIGRATION

### 1. Target Architecture Blueprint
To move away from the high-risk, manual "SSH and git pull" setup on a single server, I propose a fully automated, cloud-native architecture on **Amazon Web Services (AWS)**.

* **Compute:** **AWS ECS Fargate (Elastic Container Service).** This replaces the single server with a serverless, managed container platform. It completely removes the stress of managing underlying host operating systems, security patches, or manual SSH keys.
* **Database:** **AWS RDS PostgreSQL (Multi-AZ).** We move from a locally managed database to a fully managed RDS instance setup across multiple availability zones. This guarantees automated daily backups, point-in-time recovery, and instant failover protection.
* **Routing & Load Balancing:** **AWS Application Load Balancer (ALB).** The ALB sits in front of our ECS tasks, securely terminating SSL/TLS certificates (managed for free via AWS Certificate Manager) and evenly distributing incoming web and API traffic.

### 2. Infrastructure as Code (Terraform) Setup
To ensure our infrastructure is completely repeatable and documented, everything will be tracked via **Terraform**. We structure the configuration using a modular framework to isolate environments and prevent blast-radius issues:

* **State Management:** We will store the Terraform state file (`terraform.tfstate`) remotely in a secure **Amazon S3 bucket** with encryption turned on. To prevent two engineers from running Terraform at the exact same time and corrupting the infrastructure, we use an **Amazon DynamoDB table** for strict state locking.
* **IaC Pipeline Guardrails:** We integrate Terraform directly into our pipeline via automated infra check workflows. When an engineer opens a Pull Request modifying files inside the `/terraform` tree, the pipeline automatically triggers linting, verification, and plan mockups, posting the structural changes directly as a comment on the PR for review before execution.

### 3. Fully Automated CI/CD Lifecycle & Workflow Structure
Once the infrastructure is up, the application deployment lifecycle will be 100% automated using **GitHub Actions**. The core pipeline files live natively under the `.github/workflows/` directory.

#### Repository File & Directory Layout
To keep this challenge clean and focused on high-level judgment, the directories in this repository contain empty placeholders mapping out our complete structural layout. Here is how the infrastructure and workflow files are organized and why:

```text
├── .github/
│   └── workflows/               # Automated CI/CD Lifecycle Pipelines
│       ├── infra-pipeline.yml   # Handles Terraform Validation, Planning, and Application
│       └── app-deploy.yml       # Handles Code Testing, Docker Building, and Rolling Rollouts
├── .gitignore                   # Safety mesh; ensures state files and keys never leak to Git
├── README.md                    # Core architecture blueprint and challenge answers
└── terraform/
    ├── modules/                 # Reusable, isolated infrastructure blocks
    │   ├── networking/          # Provisions VPC, Subnets, Internet Gateways, and ALB
    │   ├── compute/             # Provisions ECR Registries and ECS Fargate Task Definitions
    │   └── database/            # Provisions the Multi-AZ AWS RDS PostgreSQL instances
    └── environments/            # Live execution spaces matching actual infrastructure environments
        └── prod/
            ├── providers.tf     # Configures AWS provider versions and global resource tagging
            ├── backend.tf       # Points to the S3 bucket and DynamoDB table for remote state locking
            ├── main.tf          # Orchestration layer; hooks modules together and passes variables
            └── variables.tf     # Explicit definition inputs used to customize production values
```

#### Detailed Execution Sequence
The `app-deploy.yml` pipeline triggers automatically on code delivery branches and executes the following steps sequentially inside isolated runner blocks:

1. **Code Push / Merge to Main:** The workflow triggers instantly when a code change is merged into the main branch.
2. **Pre-Build Code Analysis:** Runs unit tests and **Static Application Security Testing (SAST)** linters directly within the GitHub runner environment. If someone pushes broken logic or leaks an API secret, the pipeline drops immediately; this happens before wasting cloud bandwidth or container registry storage.
3. **Containerization & Git SHA Tagging:** Builds the Docker image and tags it with the immutable Git Commit SHA; you can look at any live container tag and trace it directly to the exact commit on GitHub.
4. **Secure ECR Push & Vulnerability Scan (After Image Push):** Pushes the image to a private **AWS Elastic Container Registry (ECR)**. Landing in ECR instantly triggers an **Amazon Inspector** scan to check the base OS layers for CVE vulnerabilities.
5. **Task Definition Update & Rolling Rollout:** The pipeline updates the ECS Task Definition with the new image tag. This kicks off an automated zero-downtime rolling update across our cluster nodes.

### 4. Zero-Downtime Migration Sequence
To safely transition the live app from the old single server to this new AWS setup without disrupting active property hunters across Lagos, Abuja, and Port Harcourt, we execute a meticulous 4-phase plan:

* **Phase 1: Infrastructure Provisioning & Code Verification.** Run the Terraform manifests to spin up the VPC, ALB, ECS, and RDS infrastructure. Build the initial Docker image via the new pipeline and verify that the application successfully boots inside ECS using a temporary internal testing endpoint.
* **Phase 2: Live Database Replication.** Establish a secure network connection between the old single cloud server and the new AWS RDS instance. Configure a continuous logical replication stream from the old PostgreSQL database to the new RDS database. This ensures all live listings, user profiles, and rental records stay perfectly synchronized in real-time.
* **Phase 3: The Safe Cutover Window.** Lower the Time-To-Live (TTL) settings on your public DNS records (Route 53) to 60 seconds to prepare for a fast change. During a low-traffic window (e.g., 3:00 AM WAT), place the old server's web application into a temporary "Read-Only" mode. Wait a few moments for the final database transactions to replicate fully to RDS, ensuring zero data loss.
* **Phase 4: DNS Flip & Decommission.** Point the public DNS records away from the old server's static IP address and direct them to the new AWS Application Load Balancer (ALB). Verify that live traffic is routing successfully to the healthy ECS Fargate containers. Keep the old server running as an isolated backup for 48 hours before safely shutting it down.

---

## PART B: INCIDENT SCENARIO TRIAGE (2:00 AM OUTAGE)

With 20% of search requests returning errors while the rest of the platform stays healthy, we are dealing with a **partial infrastructure failure or an asymmetric task malfunction**. I would execute the following structured isolation playbooks within the first 15 minutes:

### 1. Phase 1: Rapid Infrastructure Isolation (Minutes 1–5)
* **Check the Load Balancer Metrics:** Open the AWS CloudWatch console and view the ALB metrics. I will check if the 500-series errors are originating from a specific target node or if they are distributed evenly. If we run 5 container tasks and exactly 1 task is failing health checks or throwing errors, the ALB might still be routing some traffic to it before pulling it out of rotation.
* **Inspect Container Resource Lifecycles:** Check the ECS console for recent container restart loops. A 20% error rate can happen if a container task is crashing due to an **Out Of Memory (OOM) Kill** triggered by a heavy search query, rebooting, taking traffic, and crashing again.

### 2. Phase 2: Application Log & Database Drilldown (Minutes 5–10)
* **Query the Log Aggregator:** Check the monitoring logs (CloudWatch Logs or Elasticsearch) filtering specifically for `status=500` or `error` strings originating from the `/search` endpoint. I am looking for specific code exceptions, such as database connection timeouts or missing table indexes.
* **Analyze Database Connection Pools:** Check the RDS CPU utilization and active connection metrics. A heavy or poorly indexed search query can lock database tables or exhaust the available connection pool, causing 1 out of every 5 requests to timeout and fail while static pages continue to load smoothly from the cache.

### 3. Phase 3: Senior Escalation Framework
* **Resolution Scope:** If the root cause is infrastructure-related (e.g., a frozen container task, a stuck deployment, an easily fixable database configuration tweak, or clear resource exhaustion that can be solved by scaling up), **I will resolve it autonomously.** I will document the actions taken and update the CTO in the morning.
* **Escalation Criteria:** I will only wake up or page the engineering team if the logs conclusively show a **fatal application code defect** (such as a bad database migration script or a breaking code bug merged right before midnight) that requires a deep code fix or a product rollback that I do not have authorization to perform alone.

---

## PART C: SCALING JUDGMENT (5x TRAFFIC PROMOTION)

### What I Would Do: 
To handle the 5x surge during the promotional week, I would focus on preemptive horizontal scaling and edge caching. I would configure AWS ECS to scale out its container task count immediately to handle the increased API load, and scale up our managed RDS database instance size to a larger tier a few days before the campaign begins to provide solid database headroom. Crucially, I would implement aggressive caching rules on our Content Delivery Network (CDN / CloudFront) for all static property images, layout assets, and non-real-time search listings to ensure 80% of the traffic spikes never even hit our core servers.

### What I Would Not Deliberately Do:
I would strongly advise against any major architectural changes or last-minute software rewrites right before the campaign. I would not attempt to break the monolithic web application into microservices, switch databases, or implement complex auto-scaling algorithms that haven't been stress-tested. On a short timeline, over-engineering infrastructure introduces far more bugs and instability than a simple, predictable horizontal scale-up.
