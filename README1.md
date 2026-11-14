# End-to-End Setup for Deploying Applications with Argo CD on AWS EKS

This README provides a step-by-step guide to creating an AWS EKS cluster, installing Argo CD, deploying applications using GitOps, configuring NGINX Ingress, enabling HTTPS with Cert-Manager, and setting up Horizontal Pod Autoscaling (HPA).

## Table of Contents

1. [Create an EKS Cluster](#1-create-an-eks-cluster)
2. [Install Argo CD](#2-install-argo-cd)
3. [Configure Argo CD with EKS](#3-configure-argo-cd-with-eks)
4. [Deploy Applications Using Argo CD](#4-deploy-applications-using-argo-cd)
5. [Deploy NGINX Ingress Controller](#5-deploy-nginx-ingress-controller)
6. [Enable HTTPS Using Cert-Manager](#6-enable-https-using-cert-manager)
7. [Verify Application Deployment](#7-verify-application-deployment)
8. [Configure Autoscaling Using HPA](#8-configure-autoscaling-using-hpa)

---

## 1. Create an EKS Cluster

### Step 1: Create an EKS Cluster Without a Node Group

```bash
eksctl create cluster \
  --name=bankapp \
  --region=ap-south-1 \
  --version=1.31 \
  --without-nodegroup
```

### Step 2: Associate IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
  --region ap-south-1 \
  --cluster bankapp \
  --approve
```

### Step 3: Create a Managed Node Group

```bash
eksctl create nodegroup \
  --cluster=bankapp \
  --region=ap-south-1 \
  --name=bankapp \
  --node-type=t2.medium \
  --nodes=2 \
  --nodes-min=2 \
  --nodes-max=2 \
  --node-volume-size=29 \
  --ssh-access \
  --ssh-public-key=k8s-in-one-shot
```

### Step 4: Verify EKS Cluster

```bash
kubectl get nodes
kubectl get pods -A
```

---

## 2. Install Argo CD

### Step 1: Create Argo CD Namespace

```bash
kubectl create namespace argocd
```

### Step 2: Install Argo CD Using Official Manifest

```bash
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Step 3: Verify Argo CD Pods

```bash
watch kubectl get pods -n argocd
```

### Step 4: Install Argo CD CLI

```bash
curl --silent --location \
  -o /usr/local/bin/argocd \
  https://github.com/argoproj/argo-cd/releases/download/v2.4.7/argocd-linux-amd64

chmod +x /usr/local/bin/argocd
```

Verify installation:

```bash
argocd version
```

### Step 5: Change Argo CD Server Service to NodePort

```bash
kubectl patch svc argocd-server \
  -n argocd \
  -p '{"spec": {"type": "NodePort"}}'
```

Verify the service:

```bash
kubectl get svc -n argocd
```

### Step 6: Configure Security Group

In the AWS Console:

1. Open the EC2 Console.
2. Navigate to Security Groups.
3. Find the security group associated with your EKS worker nodes.
4. Add an inbound rule for the NodePort assigned to the `argocd-server` service.
5. Restrict access to trusted IP addresses.

### Step 7: Access Argo CD Web UI

Open the following URL in your browser:

```text
http://<public-ip-of-worker-node>:<NodePort>
```

---

## 3. Configure Argo CD with EKS

### Step 1: Retrieve the Initial Admin Password

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```

### Step 2: Login to Argo CD CLI

```bash
argocd login <public-ip-of-worker-node>:<NodePort> \
  --username admin
```

Enter the initial admin password when prompted.

### Step 3: Check Available Clusters

```bash
argocd cluster list
```

### Step 4: Get EKS Cluster Context

```bash
kubectl config get-contexts
```

### Step 5: Add EKS Cluster to Argo CD

```bash
argocd cluster add <cluster-context-name> \
  --name bankapp-eks-cluster
```

Replace `<cluster-context-name>` with your actual Kubernetes context name.

Example:

```bash
argocd cluster add Madhup@bankapp.us-west-1.eksctl.io \
  --name bankapp-eks-cluster
```

---

## 4. Deploy Applications Using Argo CD

### Step 1: Prepare Kubernetes Manifests

Create a Git repository and organize your Kubernetes manifests.

Example directory structure:

```text
bankapp/
├── namespace.yaml
├── deployment.yaml
├── service.yaml
├── ingress.yaml
└── bankapp-hpa.yml
```

### Step 2: Create an Argo CD Application

```bash
argocd app create bankapp \
  --repo <your-git-repo-url> \
  --path <path-to-manifests> \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace bankapp-namespace
```

Replace the repository URL and manifest path with your actual values.

### Step 3: Sync Application

```bash
argocd app sync bankapp
```

### Step 4: Monitor Application Status

```bash
argocd app list
argocd app get bankapp
```

---

## 5. Deploy NGINX Ingress Controller

### Step 1: Add the NGINX Ingress Helm Repository

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

### Step 2: Update Helm Repository

```bash
helm repo update
```

### Step 3: Install NGINX Ingress Controller

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

### Step 4: Verify Installation

```bash
kubectl get pods -n ingress-nginx
```

### Step 5: Retrieve Load Balancer Information

```bash
kubectl get svc -n ingress-nginx
```

Check the external address assigned to the NGINX Load Balancer.

### Step 6: Update DNS

Create or update your DNS record to point your domain to the Load Balancer address.

Example domain:

```text
junoon.trainwithshubham.com
```

---

## 6. Enable HTTPS Using Cert-Manager

### Step 1: Install Cert-Manager

```bash
kubectl apply -f \
  https://github.com/cert-manager/cert-manager/releases/download/v1.13.1/cert-manager.yaml
```

### Step 2: Create Let's Encrypt ClusterIssuer

Create a file named `letsencrypt-clusterissuer.yaml`.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

Replace `your-email@example.com` with your actual email address.

### Step 3: Apply ClusterIssuer

```bash
kubectl apply -f letsencrypt-clusterissuer.yaml
```

### Step 4: Update Ingress with TLS Configuration

Modify your Ingress manifest to:

- Enable TLS.
- Configure your application domain.
- Reference the `letsencrypt-prod` ClusterIssuer.
- Configure the TLS secret.

Apply the updated Ingress manifest:

```bash
kubectl apply -f <your-ingress-file>
```

### Step 5: Verify Certificate Issuance

```bash
kubectl get certificate -n bankapp-namespace
kubectl get clusterissuer
```

---

## 7. Verify Application Deployment

### Step 1: Check Application Resources

```bash
kubectl get all -n bankapp-namespace
```

### Step 2: Check Ingress

```bash
kubectl get ingress -n bankapp-namespace
```

### Step 3: Access the Application

Open your application in a browser:

```text
https://junoon.trainwithshubham.com
```

---

## 8. Configure Autoscaling Using HPA

### Step 1: Install Metrics Server

```bash
kubectl apply -f \
  https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Step 2: Verify Metrics Server

```bash
kubectl get deployment metrics-server -n kube-system
```

### Step 3: Check Node and Pod Metrics

```bash
kubectl top nodes
kubectl top pods -n bankapp-namespace
```

### Step 4: Apply Horizontal Pod Autoscaler

```bash
kubectl apply -f bankapp-hpa.yml
```

### Step 5: Verify HPA

```bash
kubectl get hpa -n bankapp-namespace
kubectl describe hpa -n bankapp-namespace
```

---

## Conclusion

In this project, we covered an end-to-end application deployment workflow using AWS EKS and Argo CD.

### Technologies Used

- **AWS EKS** – Managed Kubernetes Service
- **eksctl** – EKS Cluster Management
- **Argo CD** – GitOps Continuous Delivery
- **Kubernetes** – Container Orchestration
- **Helm** – Kubernetes Package Manager
- **NGINX Ingress Controller** – External Application Access
- **Cert-Manager** – HTTPS and SSL Certificate Management
- **Let's Encrypt** – Free SSL Certificates
- **Metrics Server** – Resource Metrics
- **HPA** – Horizontal Pod Autoscaling

**GitOps Workflow:**

```text
Developer
    |
    v
GitHub Repository
    |
    v
Argo CD
    |
    v
AWS EKS Cluster
    |
    v
Kubernetes Application
    |
    v
NGINX Ingress + HTTPS
    |
    v
End Users
```

This setup demonstrates how to deploy and manage Kubernetes applications on AWS EKS using GitOps practices.
