# Coworking Space Service Extension - Deployment Documentation

This repository contains the containerization and automated deployment pipeline for the **Coworking Space Analytics Service** on AWS EKS using Docker, AWS CodeBuild, AWS ECR, Kubernetes, and Helm.

---

## 1. Project Overview & Architecture

The application is a Python Flask microservice that connects to a PostgreSQL database hosted within an Amazon EKS cluster. It serves reports regarding user visits and daily workspace utilization.

### Architecture Components:
* **Containerization:** Docker multi-stage/lean build using `python:3.10-slim`.
* **Container Registry:** Amazon ECR (`coworking-analytics`).
* **CI/CD Pipeline:** AWS CodeBuild automated via `buildspec.yml`.
* **Database:** PostgreSQL provisioned via Bitnami Helm Chart with persistent data and secret management.
* **Orchestration:** Amazon EKS cluster (`my-cluster`) with Kubernetes Deployments, Services, ConfigMaps, and Secrets.
* **Monitoring & Observability:** AWS CloudWatch for CodeBuild pipeline execution and application logs.

---

## 2. Directory Structure

```text
├── analytics/
│   ├── app.py              # Flask application with reporting endpoints
│   ├── config.py           # Database connection and environment configuration
│   ├── Dockerfile          # Docker container configuration
│   └── requirements.txt    # Python dependencies
├── db/
│   ├── 1_create_tables.sql # DDL scripts for tables
│   ├── 2_seed_users.sql    # Seed data for users
│   └── 3_seed_tokens.sql   # Seed data for tokens
├── k8s/
│   ├── configmap.yaml      # Environment variables configuration
│   ├── deployment.yaml     # Kubernetes Deployment with resource limits and probes
│   └── service.yaml        # ClusterIP service exposing the application
├── prints/                 # Submission screenshots
├── buildspec.yml           # AWS CodeBuild specifications
└── README.md               # Project documentation
```

---

## 3. Project Deliverables

| Deliverable | Description | File / Location |
| :--- | :--- | :--- |
| **1. Dockerfile** | Multi-stage/lean Python base image | [`analytics/Dockerfile`](analytics/Dockerfile) |
| **2. CodeBuild Pipeline** | Successful build pipeline run | `prints/2_codebuild_pipeline.png` |
| **3. AWS ECR Repository** | ECR repository with tagged image | `prints/3_ecr_repository.png` |
| **4. `kubectl get svc`** | Services running in cluster | `prints/4_5_6_kubectl_svc_pods_database.png` |
| **5. `kubectl get pods`** | Pods in 1/1 Running state | `prints/4_5_6_kubectl_svc_pods_database.png` |
| **6. `describe svc` (DB)** | PostgreSQL service details | `prints/4_5_6_kubectl_svc_pods_database.png` |
| **7. `describe deployment`** | Application deployment details | `prints/7_kubectl_describe_deployment.png` |
| **8. Kubernetes Manifests** | ConfigMap, Deployment, Service | [`k8s/`](k8s/) |
| **9. CloudWatch Logs** | Application / build logs in CloudWatch | `prints/9_cloudwatch_logs.png` |
| **10. Documentation** | Complete technical deployment README | [`README.md`](README.md) |


---

## 4. Build & Deployment Process

### Prerequisites
* AWS CLI v2 configured with appropriate IAM credentials
* `kubectl` connected to the EKS cluster (`aws eks update-kubeconfig --name my-cluster --region us-east-1`)
* `helm` v3 installed
* `docker` installed

### Database Setup
1. Add the Bitnami repository and install PostgreSQL:
   ```bash
   helm repo add bitnami https://charts.bitnami.com/bitnami
   helm repo update
   helm install postgresql bitnami/postgresql --set primary.persistence.enabled=false
   ```
2. Retrieve the PostgreSQL password from Kubernetes Secret:
   ```bash
   export POSTGRES_PASSWORD=$(kubectl get secret --namespace default postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)
   ```
3. Seed database tables and records directly into the running database pod:
   ```bash
   kubectl exec -i postgresql-0 -- env PGPASSWORD="$POSTGRES_PASSWORD" psql -U postgres -d postgres < db/1_create_tables.sql
   kubectl exec -i postgresql-0 -- env PGPASSWORD="$POSTGRES_PASSWORD" psql -U postgres -d postgres < db/2_seed_users.sql
   kubectl exec -i postgresql-0 -- env PGPASSWORD="$POSTGRES_PASSWORD" psql -U postgres -d postgres < db/3_seed_tokens.sql
   ```

### CI/CD Pipeline (AWS CodeBuild & ECR)
1. An Amazon ECR repository `coworking-analytics` is created.
2. An AWS CodeBuild project is linked to the GitHub repository.
3. Every build triggered executes `buildspec.yml`:
   * Authenticates Docker to AWS ECR.
   * Builds the Docker image tagging with semantic versioning (`1.0.0`) and `latest`.
   * Pushes the image to Amazon ECR.

### Kubernetes Application Deployment
Apply the Kubernetes manifests:
```bash
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

Verify status:
```bash
kubectl get pods
kubectl get svc
```

### Verifying Application Endpoints
Port-forward the service to test locally:
```bash
kubectl port-forward svc/coworking-analytics 5153:5153 &

curl http://127.0.0.1:5153/api/reports/daily_usage
curl http://127.0.0.1:5153/api/reports/user_visits
```

---

## 5. How to Release New Builds

To deploy changes to production:
1. Commit and push the updated application code to the GitHub repository:
   ```bash
   git add .
   git commit -m "Release version 1.1.0"
   git push origin main
   ```
2. In **AWS CodeBuild**, trigger a new build (or let Webhooks trigger it automatically). CodeBuild will test, build, and push the new image tag to Amazon ECR.
3. Update `k8s/deployment.yaml` with the new image tag and apply:
   ```bash
   kubectl apply -f k8s/deployment.yaml
   ```
4. Perform zero-downtime rolling update:
   ```bash
   kubectl rollout status deployment coworking-analytics
   ```

---

## 6. Standout Suggestions & Architecture Answers

### 1. Reasonable Memory and CPU Allocation
The `deployment.yaml` defines explicit resource requests and limits:
* **Requests:** `cpu: 100m`, `memory: 64Mi` - Ensures Kubernetes scheduler reserves minimal necessary resources without node starvation.
* **Limits:** `cpu: 250m`, `memory: 256Mi` - Prevents any container runaway memory leak or CPU spike from impacting other pods on the same node.

### 2. Best AWS EC2 Instance Type for the Application
The `t4g.small` (or `t3.small` / `t4g.medium`) is the ideal instance type for this workload. Powered by AWS Graviton2 processors, `t4g` instances provide the best price-to-performance ratio for light, burstable web services and analytics batch processing while maintaining minimal baseline cost.

### 3. Thoughts on Cost Optimization
* **AWS Spot Instances / Karpenter:** Utilize Spot instances for EKS managed node groups for non-production environments or worker pools, saving up to 70-90% compared to On-Demand pricing.
* **Horizontal Pod Autoscaler (HPA) & Cluster Autoscaler:** Scale pods and nodes down to the minimum required during non-peak hours, avoiding idle resource waste.
* **AWS Fargate for EKS or Serverless DB:** For low-traffic internal microservices, serverless container execution eliminates fixed EC2 node baseline costs.