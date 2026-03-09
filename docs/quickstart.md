# Arkila AWS Crossplane Configuration — Quick Start Guide

This guide walks you through installing the `arkila-aws` Crossplane Configuration package and provisioning your first AWS ECR repository using a Crossplane claim.

---

## Prerequisites

Before you begin, ensure the following are in place:

| Requirement | Notes |
|-------------|-------|
| Kubernetes cluster | v1.24+ recommended |
| Crossplane | `>=v1.14.0` installed in the cluster |
| `kubectl` | Configured to target the above cluster |
| `up` CLI (optional) | Upbound CLI for pushing/pulling packages |
| AWS credentials | An IAM user or role with ECR write permissions |

### Install Crossplane (if not already installed)

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system \
  --create-namespace \
  --version 1.17.0
```

Verify Crossplane is running:

```bash
kubectl get pods -n crossplane-system
```

---

## Step 1 — Install the Configuration Package

Apply the Configuration package to your cluster. Crossplane will automatically pull and install all declared provider and function dependencies.

```bash
kubectl apply -f - <<EOF
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: arkila-aws
spec:
  package: xpkg.upbound.io/arkila/arkila-aws:v0.1.0
EOF
```

> If you are installing from a local build, use `up xpkg push` to publish the package first, then reference your registry URL above.

Monitor the installation:

```bash
kubectl get configuration arkila-aws
kubectl get configurationrevision
```

Wait until `HEALTHY` and `INSTALLED` are both `True`:

```
NAME         INSTALLED   HEALTHY   PACKAGE                                  AGE
arkila-aws   True        True      xpkg.upbound.io/arkila/arkila-aws:v0.1.0  2m
```

---

## Step 2 — Configure AWS Provider Credentials

Each tribe/environment combination needs a `ProviderConfig`. The provider config name must follow the pattern `{tribe}-{environment}` (e.g., `bg-dev`, `dcp-prod`).

### 2a. Create a Kubernetes Secret with AWS credentials

```bash
kubectl create secret generic aws-creds \
  --namespace crossplane-system \
  --from-literal=creds="[default]
aws_access_key_id = YOUR_ACCESS_KEY_ID
aws_secret_access_key = YOUR_SECRET_ACCESS_KEY"
```

> For production use, prefer IRSA (IAM Roles for Service Accounts) over static credentials.

### 2b. Create ProviderConfigs for each tribe/environment

Create a `ProviderConfig` for every `{tribe}-{environment}` combination your teams will use. Example for the Base Growth tribe in the dev environment:

```bash
kubectl apply -f - <<EOF
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: bg-dev
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-creds
      key: creds
EOF
```

Repeat for additional combinations (e.g., `bg-prod`, `ds-dev`, `dcp-prod`, etc.).

---

## Step 3 — Create an EnvironmentConfig

`EnvironmentConfig` objects supply shared, environment-specific values (region, tribe, environment, etc.) to compositions. The label `name` must match the value you will set in `spec.environmentConfig` of your claims.

```bash
kubectl apply -f - <<EOF
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: EnvironmentConfig
metadata:
  name: de-tags
  labels:
    name: de-tags
data:
  region: eu-west-1
  environment: dev
  tribe: bg
  project: myproject
  crossplane: "true"
EOF
```

Verify it was created:

```bash
kubectl get environmentconfig de-tags
```

---

## Step 4 — Provision an ECR Repository

### 4a. Create the namespace for your team

```bash
kubectl create namespace crossplane-system
# (already exists if Crossplane is installed — you can use any namespace)
```

### 4b. Apply a RepositoryClaim

```bash
kubectl apply -f - <<EOF
apiVersion: aws.arkila.com/v1alpha1
kind: RepositoryClaim
metadata:
  name: my-app-repository
  namespace: crossplane-system
spec:
  compositionRef:
    name: ecr-composition
  deletionPolicy: Delete
  environmentConfig: de-tags
  project: myproject
  owner: "Platform Engineering"
  createdby: "Jane Doe"
  squad: "DevSecOps Platforms"
  # ECR settings
  ecrName: my-app
  ecrEncryptionType: AES256
  ecrScanOnPush: true
  ecrImageTagMutability: IMMUTABLE
EOF
```

Or apply the bundled example directly:

```bash
kubectl apply -f example/ecr-repository-claim.yaml
```

### 4c. Watch the provisioning

```bash
# Check the claim status
kubectl get repositoryclaim my-app-repository -n crossplane-system

# Describe for detailed events and conditions
kubectl describe repositoryclaim my-app-repository -n crossplane-system

# Watch the underlying managed resource
kubectl get repository.ecr.aws.upbound.io
```

A successfully provisioned repository will show:

```
NAME                  SYNCED   READY   CONNECTION-SECRET   AGE
my-app-repository     True     True                        3m
```

### 4d. Retrieve status outputs

Once `READY=True`, retrieve the ECR ARN and repository URL:

```bash
kubectl get repositoryclaim my-app-repository -n crossplane-system \
  -o jsonpath='{.status.ecrArn}'

kubectl get repositoryclaim my-app-repository -n crossplane-system \
  -o jsonpath='{.status.ecrRepositoryUrl}'
```

Example output:

```
arn:aws:ecr:eu-west-1:123456789012:repository/my-app
123456789012.dkr.ecr.eu-west-1.amazonaws.com/my-app
```

---

## Step 5 — Push a Docker Image (Optional Verification)

```bash
# Authenticate Docker to ECR
aws ecr get-login-password --region eu-west-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.eu-west-1.amazonaws.com

# Tag your image
docker tag my-app:latest \
  123456789012.dkr.ecr.eu-west-1.amazonaws.com/my-app:latest

# Push
docker push 123456789012.dkr.ecr.eu-west-1.amazonaws.com/my-app:latest
```

---

## Step 6 — Cleaning Up

To delete the repository, either delete the claim (behavior depends on `deletionPolicy`):

```bash
kubectl delete repositoryclaim my-app-repository -n crossplane-system
```

- **`deletionPolicy: Delete`**: Crossplane will delete the ECR repository in AWS.
- **`deletionPolicy: Orphan`** (default): The Kubernetes objects are removed but the ECR repository remains in AWS.

---

## Troubleshooting

### Claim stuck in `Waiting` or `Unready`

```bash
kubectl describe repositoryclaim <name> -n <namespace>
```

Look for events mentioning missing `EnvironmentConfig`, unresolved `ProviderConfig`, or missing permissions.

### ProviderConfig not found

Ensure the ProviderConfig name matches the pattern `{tribe}-{environment}` exactly, and that it references valid AWS credentials.

```bash
kubectl get providerconfig
```

### EnvironmentConfig not matched

The composition selects the `EnvironmentConfig` by the label `name`. Verify:

```bash
kubectl get environmentconfig -l name=<your-environmentConfig-value>
```

### View composition pipeline logs

```bash
# Find the function pod
kubectl get pods -n crossplane-system -l pkg.crossplane.io/revision

# Check function-patch-and-transform logs
kubectl logs -n crossplane-system \
  -l pkg.crossplane.io/function=function-patch-and-transform
```

### Provider not installed

```bash
kubectl get providers
kubectl describe provider provider-aws-ecr
```

---

## Configuration Reference Summary

| Spec Field | Required | Default | Valid Values |
|------------|----------|---------|--------------|
| `environmentConfig` | No | — | Any `EnvironmentConfig` label name |
| `deletionPolicy` | No | `Orphan` | `Delete`, `Orphan` |
| `ecrName` | **Yes** | — | Any valid ECR repository name |
| `ecrEncryptionType` | No | `AES256` | `AES256`, `KMS` |
| `ecrScanOnPush` | No | `false` | `true`, `false` |
| `ecrImageTagMutability` | No | `IMMUTABLE` | `IMMUTABLE`, `MUTABLE` |
| `project` | No | — | Free text |
| `owner` | No | — | Free text |
| `createdby` | No | — | Free text |
| `squad` | No | — | Free text |

---

## Next Steps

- Read the full [Technical Documentation](./technical-documentation.md) for composition internals, tagging strategy, and provider config conventions.
- Add more compositions (S3, RDS, EKS, etc.) by following the same XRD + Composition pattern used for ECR.
- Set up CI/CD to automatically build and publish the package to the Upbound Registry on every merge to `main`.
