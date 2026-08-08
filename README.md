
<<<<<<< HEAD
This guide walks you through creating an Amazon EKS (Elastic Kubernetes Service) cluster

<<<<<<< HEAD

## Step 1: Set Up Prerequisites

Install these tools on your machine:

1.  **AWS CLI** — lets you talk to AWS from the terminal
    - Install: [https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
    - 1.  - Verify: `aws --version`
2.  **kubectl** — lets you talk to your Kubernetes cluster
    - Install: [https://kubernetes.io/docs/tasks/tools/](https://kubernetes.io/docs/tasks/tools/)
    - Verify: `kubectl version --client`
3.  **eksctl** — the official CLI tool to create EKS clusters easily
    - Install: [https://eksctl.io/installation/](https://eksctl.io/installation/)
    - Verify: `eksctl version`
4.  **Docker** — needed to build container images
    - Install: [https://docs.docker.com/get-docker/](https://docs.docker.com/get-docker/)

## Step 2: Create an AWS Account & IAM User

1.  Sign up at [https://aws.amazon.com](https://aws.amazon.com) if you don't have an account.
2.  Don't use your root account for daily work. Instead, create an IAM user:
    - Go to **IAM → Users → Create user**
    - Give it **AdministratorAccess**
    - Generate an **Access Key ID** and **Secret Access Key**
3.  Configure the AWS CLI with these credentials:

bash

```bash
   aws configure
```

Enter your Access Key, Secret Key, default region (e.g., `ap-south-1`), and output format (`json`).

## Step 3: Create the EKS Cluster

The easiest way is `eksctl`, which handles the VPC, subnets, and networking for you automatically.

bash

```bash
eksctl create cluster \
  --name demo-cluster \
  --region ap-south-1 \
  --nodegroup-name demo-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
```

What this does:

- `--name` — names your cluster
- `--region` — AWS region to deploy in
- `--node-type` — EC2 instance type for worker nodes
- `--nodes` — number of worker nodes to start with
- `--managed` — uses EKS Managed Node Groups (AWS handles node lifecycle for you)

⏱️ This takes **15–20 minutes**. `eksctl` is creating a VPC, subnets, IAM roles, the EKS control plane, and the worker nodes — all in one command.

## Step 4: Connect kubectl to Your Cluster

`eksctl` usually configures this automatically, but to do it manually or reconnect later:

bash

```bash
aws eks update-kubeconfig --region ap-south-1 --name demo-cluster
```

Verify the connection:

bash

```bash
kubectl get nodes
```

You should see your worker nodes listed with status `Ready`.

## Step 5: Create an ECR Repository (for your Docker images)

Your app's Docker image needs somewhere to live before Kubernetes can pull it.

bash

```bash
aws ecr create-repository --repository-name frontend --region ap-south-1
```

Note the repository URI returned (e.g., `123456789012.dkr.ecr.ap-south-1.amazonaws.com/frontend`) — you'll need it later.

## Step 6: Build, Tag, and Push Your Docker Image

bash

```bash
# Authenticate Docker to ECR
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS --password-stdin 123456789012.dkr.ecr.ap-south-1.amazonaws.com

# Build the image
docker build -t frontend .

# Tag it for ECR
docker tag frontend:latest 123456789012.dkr.ecr.ap-south-1.amazonaws.com/frontend:latest

# Push it
docker push 123456789012.dkr.ecr.ap-south-1.amazonaws.com/frontend:latest
```

Replace `123456789012` with your actual AWS account ID.

## Step 7: Install the AWS Load Balancer Controller (for Ingress)

If your app needs an Ingress (like the ALB Ingress in EKS-Demo), you need this controller installed on the cluster first.

1.  Create an IAM OIDC provider for the cluster:

bash

```bash
   eksctl utils associate-iam-oidc-provider --cluster demo-cluster --approve
```

2.  Create the IAM policy (using the `iam_policy.json` from the repo, or download AWS's official one):

bash

```bash
   aws iam create-policy \
     --policy-name AWSLoadBalancerControllerIAMPolicy \
     --policy-document file://iam_policy.json
```

3.  Create a service account with that policy attached:

bash

```bash
   eksctl create iamserviceaccount \
     --cluster demo-cluster \
     --namespace kube-system \
     --name aws-load-balancer-controller \
     --attach-policy-arn arn:aws:iam::123456789012:policy/AWSLoadBalancerControllerIAMPolicy \
     --approve
```

4.  Install the controller via Helm:

bash

```bash
   helm repo add eks https://aws.github.io/eks-charts
   helm repo update
   helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
     -n kube-system \
     --set clusterName=demo-cluster \
     --set serviceAccount.create=false \
     --set serviceAccount.name=aws-load-balancer-controller
```

## Step 8: Deploy Your App

Update the image reference in `K8s.yaml` to point to your ECR repo, then apply it:

bash

```bash
kubectl apply -f K8s.yaml
```

Check that everything came up:

bash

```bash
kubectl get pods -n development
kubectl get svc -n development
kubectl get ingress -n development
```

## Step 9: Access Your App

Get the ALB's public address:

bash

```bash
kubectl get ingress frontend-ingress -n development
```

Copy the `ADDRESS` value and open it in your browser (it may take a few minutes for the ALB to become active).

## Step 10: Clean Up (Important — Avoid Ongoing Charges)

When you're done experimenting, delete everything to avoid being billed for idle resources:

bash

```bash
kubectl delete -f K8s.yaml
eksctl delete cluster --name demo-cluster --region ap-south-1
```

---

## Common Beginner Pitfalls

- **Forgetting to delete the cluster** — EKS clusters cost money even when idle (~$0.10/hour for the control plane alone, plus EC2 node costs).
- **Wrong region** — make sure your CLI region, ECR repo, and cluster are all in the same region.
- **IAM permission errors** — if `eksctl` or `kubectl` fails with access denied, double-check your IAM user's permissions.
- **Image pull errors** — if pods show `ImagePullBackOff`, verify the image URI in `K8s.yaml` matches your ECR repo exactly, and that your nodes have permission to pull from ECR (managed node groups get this by default).
- **Ingress stuck with no address** — this usually means the AWS Load Balancer Controller isn't installed or the IAM policy/service account isn't set up correctly (Step 7).

## Useful Commands Cheat Sheet

| Task                        | Command                                          |
| --------------------------- | ------------------------------------------------ |
| List clusters               | `eksctl get cluster`                             |
| Check nodes                 | `kubectl get nodes`                              |
| Check pods                  | `kubectl get pods -n <namespace>`                |
| View pod logs               | `kubectl logs <pod-name> -n <namespace>`         |
| Describe a resource (debug) | `kubectl describe pod <pod-name> -n <namespace>` |
| Delete cluster              | `eksctl delete cluster --name <name>`            |
=======
>>>>>>> origin
