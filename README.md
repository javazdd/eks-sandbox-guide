# EKS + Observability Pipelines Worker Setup Guide

Step-by-step guide for deploying an AWS EKS cluster and running the Datadog Observability Pipelines Worker on it.

---

## Prerequisites

### Tools Required

| Tool | Purpose |
|------|---------|
| `awscli` | AWS command-line interface |
| `aws-vault` | Securely manage AWS credentials |
| `eksctl` | EKS cluster management |
| `kubectl` | Kubernetes CLI |
| `helm` | Deploy the OPW Helm chart |

### Install via Homebrew

```bash
brew install awscli aws-vault eksctl helm
```

---

## AWS Access Setup

### 1. Configure AWS SSO

> **Note:** The `sso_start_url` must be the full AWS portal URL, not a shortlink. To resolve one:
> ```bash
> curl -sI https://dtdg.co/aws-sso-prod | grep -i location
> # → https://d-906757b57c.awsapps.com/start
> ```

Add profiles to `~/.aws/config`:

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
> TOKEN=$(python3 -c "
> import json, glob, os
> files = glob.glob(os.path.expanduser('~/.aws/sso/cache/*.json'))
> for f in files:
>     d = json.load(open(f))
>     if 'accessToken' in d:
>         print(d['accessToken'])
>         break
> ")
> aws sso list-accounts --access-token "$TOKEN" --region us-east-1
> aws sso list-account-roles --access-token "$TOKEN" --account-id <ACCOUNT_ID> --region us-east-1
> ```

### 2. Log in via SSO

```bash
aws sso login --profile ese-sandbox
```

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
# Look for * next to confused-country-ladybug
```

### 3. Verify Cluster Access

```bash
aws-vault exec ese-sandbox -- kubectl get nodes
```

---

## Adding a Node Group

A freshly created EKS cluster has no worker nodes. Create a `nodegroup.yaml` with the cluster's VPC details:

> **Tip:** Get the VPC/subnet info from the existing cluster:
> ```bash
> aws-vault exec ese-sandbox -- aws eks describe-cluster --name <CLUSTER> --region <REGION> \
>   --query 'cluster.{vpcId:resourcesVpcConfig.vpcId, subnets:resourcesVpcConfig.subnetIds, sg:resourcesVpcConfig.clusterSecurityGroupId}'
>
> aws-vault exec ese-sandbox -- aws ec2 describe-subnets --subnet-ids <IDS> --region <REGION> \
>   --query 'Subnets[].{id:SubnetId,az:AvailabilityZone,public:MapPublicIpOnLaunch}'
> ```

```yaml
# nodegroup.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: confused-country-ladybug
  region: us-east-2
  version: "1.35"

vpc:
  id: vpc-02cd03d9675f220e6
  securityGroup: sg-0596acf76d52315e1
  subnets:
    private:
      us-east-2a:
        id: subnet-0c80385f5ea9602dc
      us-east-2b:
        id: subnet-0f3378be916ce7762
      us-east-2c:
        id: subnet-03954e16f5f9ef86a

managedNodeGroups:
  - name: standard
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 0
    maxSize: 3
    volumeSize: 30
    privateNetworking: true
    subnets:
      - subnet-0c80385f5ea9602dc
      - subnet-0f3378be916ce7762
      - subnet-03954e16f5f9ef86a
```

```bash
aws-vault exec ese-sandbox -- eksctl create nodegroup -f nodegroup.yaml
```

> **Note:** If eksctl times out, the node group may still have been created successfully — check with `kubectl get nodes`.

---

## Deploy the Observability Pipelines Worker

### 1. Add the Datadog Helm repo

```bash
helm repo add datadog https://helm.datadoghq.com
helm repo update
```

### 2. Create your values file

```yaml
# op-values.yaml
datadog:
  apiKey: "<DD_API_KEY>"
  pipelineId: "<DD_OP_PIPELINE_ID>"
  site: datadoghq.com

replicas: 1
```

> Find your Pipeline ID in the Datadog UI under **Observability Pipelines → your pipeline**.

### 3. Install the Helm chart

```bash
aws-vault exec ese-sandbox -- helm install opw datadog/observability-pipelines-worker \
  -f op-values.yaml
```

### 4. Verify the worker is running

```bash
aws-vault exec ese-sandbox -- kubectl get pods -l app.kubernetes.io/instance=opw
# Expected: opw-observability-pipelines-worker-0   1/1   Running
```

---

## Useful Aliases

```bash
alias av='aws-vault exec ese-sandbox --'
alias avk='aws-vault exec ese-sandbox -- kubectl'
```

```bash
avk get pods
avk get nodes
av helm list
```

---

## Scaling Node Groups

```bash
# List node groups
aws-vault exec ese-sandbox -- eksctl get nodegroup \
  --cluster confused-country-ladybug --region us-east-2

# Scale down to 0 (save costs when not in use)
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

```bash
# Uninstall OPW
aws-vault exec ese-sandbox -- helm uninstall opw

# Delete the cluster
aws-vault exec ese-sandbox -- eksctl delete cluster --name confused-country-ladybug --region us-east-2
```

> AWS Sandbox resources are automatically cleaned up after 90 days.
