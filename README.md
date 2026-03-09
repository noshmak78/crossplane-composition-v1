# Arkila AWS Crossplane Configuration

A Crossplane Configuration package that provides production-grade, opinionated abstractions over AWS resources for Arkila engineering teams. Platform teams define Compositions and XRDs; application teams consume them via simple Claims.

## What's included

| Resource | API | Composition |
|----------|-----|-------------|
| ECR Repository | `aws.arkila.com/v1alpha1` `RepositoryClaim` | `ecr-composition` |

## Requirements

- Crossplane `>= v1.14.0`
- `kubectl` configured against your cluster

## Quick start

See [docs/quickstart.md](docs/quickstart.md) for a full walkthrough. The short version:

```bash
# 1. Install the configuration package
kubectl apply -f - <<EOF
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: arkila-aws
spec:
  package: xpkg.upbound.io/arkila/arkila-aws:v0.1.0
EOF

# 2. Create an AWS credentials secret
kubectl create secret generic aws-creds \
  --namespace crossplane-system \
  --from-literal=creds="[default]
aws_access_key_id = YOUR_ACCESS_KEY_ID
aws_secret_access_key = YOUR_SECRET_ACCESS_KEY"

# 3. Create a ProviderConfig (name = {tribe}-{environment})
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

# 4. Create an EnvironmentConfig
kubectl apply -f - <<EOF
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: EnvironmentConfig
metadata:
  name: bg-tags
  labels:
    name: bg-tags
data:
  region: eu-west-1
  environment: dev
  tribe: bg
  project: myproject
  crossplane: "true"
EOF

# 5. Claim an ECR repository
kubectl apply -f example/ecr-repository-claim.yaml
```

## Tribes

The provider config and tags are resolved per tribe/environment combination. Valid tribe keys:

| Key | Name |
|-----|------|
| `bg` | Base Growth |
| `dmg` | Digital Marketing and Growth |
| `dcp` | Digital Channels Payment |
| `ds` | Digital Services |

Provider config naming pattern: `{tribe}-{environment}` (e.g., `bg-dev`, `dcp-prod`).

## Environments

| Key | Display Name |
|-----|-------------|
| `dev` | Development |
| `staging` | Staging |
| `uat` | UAT |
| `sandbox` | Sandbox |
| `prod` | Production |

## Tagging

All resources are automatically tagged with:

`Name` · `Environment` · `Tribe` · `Project` · `Squad` · `Owner` · `CreatedBy` · `Crossplane` · `CRQ`

Tags are sourced from a combination of the `EnvironmentConfig` (region, environment, tribe) and the Claim spec (project, squad, owner, createdby, crq).

## Dependencies

**Functions**

| Function | Version |
|----------|---------|
| `function-environment-configs` | `v0.4.0` |
| `function-patch-and-transform` | `v0.8.2` |
| `function-go-templating` | `v0.11.0` |
| `function-auto-ready` | `v0.5.2` |

**Providers**

| Provider | Version |
|----------|---------|
| `provider-aws-ecr` | `v1.23.2` |

## Repository structure

```
crossplane-composition-v1/
├── config/
│   ├── environmentconfig.yaml     # Sample EnvironmentConfig
│   └── providerconfig.yaml        # Sample ProviderConfig
├── example/
│   └── ecr-repository-claim.yaml  # Example RepositoryClaim
├── package/
│   ├── crossplane.yaml            # Package metadata & dependencies
│   └── ecr-repository/
│       ├── composition.yaml       # ECR Composition pipeline
│       └── definition.yaml        # ECR XRD schema
├── OWNERS
└── README.md
```

## Documentation

- [Quick Start Guide](docs/quickstart.md)
- [Technical Documentation](docs/technical-documentation.md)
