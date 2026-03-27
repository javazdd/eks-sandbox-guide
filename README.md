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

You need access to the Datadog ESE sandbox AWS account via AWS SSO.

### 1. Configure AWS SSO

> **Note:** The `sso_start_url` must be the full AWS portal URL — not a shortlink. To resolve a shortlink:
> ```bash
> curl -sI https://dtdg.co/aws-sso-prod | grep -i location
> # → https://d-906757b57c.awsapps.com/start
> ```

Add the following profiles to `~/.aws/config`:

```ini
[profile ese-sandbox]
sso_start_url = https://d-906757b57c.awsapps.com/start
sso_region = us-east-1
sso_account_id = 770341584863
sso_role_name = account-admin
region = us-east-2
output = json

[profile tse-playground]
sso_start_url = https://d-906757b57c.awsapps.com/start
sso_region = us-east-1
sso_account_id = 570690476889
sso_role_name = power-user
region = us-east-2
output = json
```

> **Tip:** To discover what accounts and roles you have access to after logging in:
> ```bash
> # List accounts
> TOKEN=$(python3 -c "import json,glob; f=sorted(glob.glob(os.path.expanduser('~/.aws/sso/cache/*.json')))[0]; d=json.load(open(f)); print(d.get('accessToken',''))")
> aws sso list-accounts --access-token "$TOKEN" --region us-east-1
> # List roles for an account
> aws sso list-account-roles --access-token "$TOKEN" --account-id <ACCOUNT_ID> --region us-east-1
> ```

### 2. Log in via SSO

```bash
aws sso login --profile ese-sandbox
```

This opens a browser for you to authorize. Approve the request.

### 3. Verify Access

```bash
aws-vault exec ese-sandbox -- aws eks list-clusters --region us-east-2
```

---

## Connect to an Existing EKS Cluster

### Cluster Details

- **Cluster name:** `confused-country-ladybug`
- **Region:** `us-east-2`
- **Account:** `ese-sandbox` (770341584863)
- **Kubernetes version:** 1.35

### 1. Update kubeconfig

```bash
aws-vault exec ese-sandbox -- aws eks --region us-east-2 update-kubeconfig --name confused-country-ladybug
```

### 2. Verify the Context

```bash
kubectl config get-contexts
```

Look for `*` next to the `confused-country-ladybug` context. If it's not active:

```bash
kubectl config use-context <context-name>
```

### 3. Verify Cluster Access

```bash
aws-vault exec ese-sandbox -- kubectl get nodes
```

---

## Adding a Node Group

A freshly created EKS cluster has no worker nodes. You must add a node group before deploying any workloads.

```bash
aws-vault exec ese-sandbox -- eksctl create nodegroup \
  --cluster confused-country-ladybug \
  --region us-east-2 \
  --name standard \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 0 \
  --nodes-max 3 \
  --node-volume-size 30 \
  --managed
```

> This takes 5–10 minutes. Once complete, verify with:
> ```bash
> aws-vault exec ese-sandbox -- kubectl get nodes
> ```

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
aws-vault exec ese-sandbox -- kubectl apply -f deployment.yaml
```

Verify:

```bash
aws-vault exec ese-sandbox -- kubectl get pods
```

### Option B: Helm

```bash
aws-vault exec ese-sandbox -- helm install my-release my-chart/
```

---

## Useful Aliases

Add these to your `~/.zshrc` or `~/.bashrc`:

```bash
alias av='aws-vault exec ese-sandbox --'
alias avk='aws-vault exec ese-sandbox -- kubectl'
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

## Scaling Node Groups

Scale down when not in use to save costs:

```bash
# List node groups
aws-vault exec ese-sandbox -- eksctl get nodegroup \
  --cluster confused-country-ladybug --region us-east-2

# Scale down to 0
aws-vault exec ese-sandbox -- eksctl scale nodegroup \
  --name standard --nodes=0 \
  --region us-east-2 --cluster confused-country-ladybug

# Scale back up
aws-vault exec ese-sandbox -- eksctl scale nodegroup \
  --name standard --nodes=2 \
  --region us-east-2 --cluster confused-country-ladybug
```

---

## Tear Down

**Always delete your cluster when done** to free up AWS quotas:

```bash
aws-vault exec ese-sandbox -- eksctl delete cluster --name confused-country-ladybug --region us-east-2
```

> AWS Sandbox resources are automatically cleaned up after 90 days.
