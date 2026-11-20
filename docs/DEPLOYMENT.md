# Deployment Guide

This guide covers local execution and the AWS EKS deployment. AWS resources can incur charges; destroy unused infrastructure using the Terraform repository's documented cleanup process.

## Prerequisites

For local execution:

- Docker Engine with the Docker Compose plugin
- Git

For AWS deployment:

- AWS CLI
- Terraform
- `kubectl`
- `eksctl`, Helm, and an AWS account with the IAM permissions needed for EKS and the AWS Load Balancer Controller

## Run Locally

From the repository root:

```bash
make start
```

Open the application at [http://localhost:8080](http://localhost:8080). The full upstream demo also exposes its bundled local Jaeger, Grafana, and load-generator pages through the frontend proxy.

For the minimal Compose configuration:

```bash
make start-minimal
```

Stop the Compose environment with:

```bash
make stop
```

## Provision AWS Infrastructure

Use the [Terraform infrastructure repository](https://github.com/adityasingh307/three-tier-devops-infrastructure).

The repository README describes creating its remote state backend first, then initializing and applying the EKS infrastructure:

```bash
cd backend
terraform init
terraform plan
terraform apply

cd ..
terraform init
terraform fmt -recursive
terraform validate
terraform plan
terraform apply
```

Configure AWS CLI credentials if they are not already available to your shell:

```bash
aws configure
```

Update kubeconfig using the cluster and region configured in Terraform:

```bash
aws eks --region <your-region> update-kubeconfig --name <your-cluster-name>
kubectl get nodes
```

The current Terraform README uses `us-west-2` and `my-eks-cluster`; use the values for your deployment.

## Install the AWS Load Balancer Controller

The controller needs an IAM policy and service account, then a Helm installation in `kube-system`. Follow the current [AWS EKS installation guide](https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html) for the policy, IAM role, and chart version required by your cluster. Verify readiness with:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

## Deploy the Application

Apply the combined demo manifest, then apply the Product Catalog manifest so the deployment uses the image managed by this repository's CI workflow. Apply the Ingress after the controller is ready:

```bash
kubectl apply -f kubernetes/complete-deploy.yaml
kubectl apply -f kubernetes/productcatalog/
kubectl apply -f kubernetes/frontendproxy/ingress.yaml
```

Check workloads and wait for the Ingress to receive an AWS hostname:

```bash
kubectl get deployments
kubectl get pods
kubectl get ingress
```

Open the ALB hostname shown by `kubectl get ingress` in a browser.

## Configure Product Catalog CI/CD

In the GitHub repository settings, add these Actions secrets:

- `DOCKERHUB_USERNAME`
- `DOCKERHUB_TOKEN`
- `TOKEN` with permission to push the workflow's manifest update

The workflow is scoped to changes under `src/product-catalog/**`. Pull requests run validation and build the image without publishing. A push to `main` publishes an image tagged with the Actions run ID and updates `kubernetes/productcatalog/deploy.yaml`.

## Configure Argo CD

Install Argo CD by following its [Getting Started guide](https://argo-cd.readthedocs.io/en/stable/getting_started/). Create an application that tracks:

- Repository: `https://github.com/adityasingh307/k8s-microservies-devops-project.git`
- Branch: `main`
- Path: `kubernetes/productcatalog`
- Destination namespace: `default`
- Sync policy: automated

The Argo CD application shown in the deployment screenshot is named `product-catalog-server`. Automated sync lets Argo CD apply the manifest update committed by GitHub Actions. See the [Argo CD automated sync documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/).

## Clean Up

When the AWS environment is no longer needed, remove the Kubernetes Ingress while the Load Balancer Controller is available so it can clean up the ALB. Then follow the Terraform repository's cleanup instructions from its infrastructure root. Its README notes that the remote state backend is managed separately and is not removed by the infrastructure destroy.
