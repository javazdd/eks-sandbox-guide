# EKS Sandbox Setup Guide

Step-by-step guide for connecting to and deploying containers on an AWS EKS cluster using `aws-vault`, `eksctl`, and `kubectl`.

---

## Prerequisites

### Tools Required

| Tool | Purpose |
|------|---------|
| `awscli` | AWS command-line interface |
| `aws-vault` | Securely manage AWS credentials |
| `eksctl` | EKS cluster management |
| `kubectl` | Kubernetes CLI |

### Install via Homebrew

```bash
brew install awscli aws-vault eksctl
```

Verify installations:

```bash
aws --version
aws-vault --version
eksctl version
kubectl version --client
```

---

## AWS Access Setup

You need access to the Datadog TSE sandbox AWS account (ID: `659775407889`) via AWS SSO.

### 1. Configure AWS SSO

Follow internal setup docs to get access to the `sso-tse-sandbox-account-admin` profile.

Your `~/.aws/config` should include a block like:

```ini
[profile sso-tse-sandbox-account-admin]
sso_start_url = https://dtdg.co/aws-sso-prod
sso_region = us-east-1
sso_account_id = 659775407889
sso_role_name = AdministratorAccess
region = us-east-2
```

### 2. Verify AWS Vault Connection

```bash
aws-vault exec sso-tse-sandbox-account-admin -- aws s3 ls
```

This will open a browser prompt to authorize. On success, you'll see a list of S3 buckets.

---

## Connect to an Existing EKS Cluster

### Cluster Details (this guide)

- **Cluster name:** `confused-country-ladybug`
- **Region:** `us-east-2`

### 1. Update kubeconfig

```bash
aws-vault exec sso-tse-sandbox-account-admin -- \
  aws eks --region us-east-2 update-kubeconfig --name confused-country-ladybug
```

This adds the cluster context to your local `~/.kube/config`.

### 2. Verify the Context

```bash
kubectl config get-contexts
```

Look for `*` next to `confused-country-ladybug`. If it's not active:

```bash
kubectl config use-context <context-name>
```

### 3. Verify Cluster Access

```bash
aws-vault exec sso-tse-sandbox-account-admin -- kubectl get nodes
```

You should see your worker nodes listed with `Ready` status.

---

## Deploying Containers

### Option A: kubectl apply

Create a manifest file (e.g., `deployment.yaml`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: nginx:latest
          ports:
            - containerPort: 80
```

Apply it:

```bash
aws-vault exec sso-tse-sandbox-account-admin -- kubectl apply -f deployment.yaml
```

Verify:

```bash
aws-vault exec sso-tse-sandbox-account-admin -- kubectl get pods
```

### Option B: Helm

```bash
aws-vault exec sso-tse-sandbox-account-admin -- helm install my-release my-chart/
```

---

## Useful Aliases

Add these to your `~/.zshrc` or `~/.bashrc` to avoid typing the full `aws-vault` prefix every time:

```bash
alias av='aws-vault exec sso-tse-sandbox-account-admin --'
alias avk='aws-vault exec sso-tse-sandbox-account-admin -- kubectl'
```

Then reload:

```bash
source ~/.zshrc
```

Usage:

```bash
avk get pods
avk get nodes
av helm install ...
av eksctl get nodegroup --cluster confused-country-ladybug --region us-east-2
```

---

## Creating a New Cluster (optional)

### 1. Install eksctl

```bash
brew install eksctl
```

### 2. Create a cluster.yaml

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: FirstnameLastnameSandbox
  region: us-east-2
  tags:
    creator: firstname.lastname
    team: technical-support-engineer
  version: "1.29"
managedNodeGroups:
  - name: standard
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 0
    volumeSize: 30
```

### 3. Create the cluster

```bash
aws-vault exec sso-tse-sandbox-account-admin -- eksctl create cluster -f cluster.yaml
```

> **Note:** If you hit quota errors (e.g., EIP limits), try a different region.

---

## Scaling Node Groups

Scale down when not in use to save costs:

```bash
# List node groups
aws-vault exec sso-tse-sandbox-account-admin -- \
  eksctl get nodegroup --cluster confused-country-ladybug --region us-east-2

# Scale down to 0
aws-vault exec sso-tse-sandbox-account-admin -- \
  eksctl scale nodegroup --name standard --nodes=0 \
  --region us-east-2 --cluster confused-country-ladybug
```

---

## Tear Down

**Always delete your cluster when done** to free up AWS quotas:

```bash
aws-vault exec sso-tse-sandbox-account-admin -- eksctl delete cluster -f cluster.yaml
```

> AWS Sandbox resources are automatically cleaned up after 90 days.
