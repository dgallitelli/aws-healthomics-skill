# ECR Pull-Through Cache Setup

Configure ECR to automatically fetch containers from public registries for HealthOmics.

## Overview

HealthOmics requires containers from private ECR with proper permissions. Two approaches:

1. **Pull-Through Cache** - Auto-fetch from Docker Hub, Quay.io, ECR Public
2. **Manual Clone** - Copy specific containers to ECR

## Quick Start

### Option 1: Pull-Through Cache (Recommended)

```
# Create cache rule for Docker Hub
CreateECRPullThroughCache(
    registry="docker.io",
    ecr_prefix="docker-hub",
    credential_arn="arn:aws:secretsmanager:...:docker-hub-creds"
)

# Create cache rule for Quay.io
CreateECRPullThroughCache(
    registry="quay.io",
    ecr_prefix="quay"
)

# Create cache rule for ECR Public
CreateECRPullThroughCache(
    registry="public.ecr.aws",
    ecr_prefix="ecr-public"
)
```

### Option 2: Clone Specific Container

```
CloneContainerToECR(
    source_uri="quay.io/biocontainers/bwa:0.7.17",
    target_repository="my-workflow/bwa",
    target_tag="0.7.17"
)
```

## Detailed Setup

### Step 1: Validate Current Configuration

```
ValidateECRConfiguration()
```

Check for existing rules and issues.

### Step 2: Create Docker Hub Secret

Docker Hub requires authentication. Create secret in Secrets Manager:

```bash
aws secretsmanager create-secret \
    --name docker-hub-creds \
    --secret-string '{"username":"myuser","accessToken":"dckr_pat_xxx"}'
```

### Step 3: Create Pull-Through Rules

```
CreateECRPullThroughCache(
    registry="docker.io",
    ecr_prefix="docker-hub",
    credential_arn="arn:aws:secretsmanager:REGION:ACCOUNT:secret:docker-hub-creds"
)
```

Supported registries:
- `docker.io` (Docker Hub)
- `quay.io` (Quay)
- `public.ecr.aws` (ECR Public)
- `ghcr.io` (GitHub Container Registry)
- `registry.k8s.io` (Kubernetes)

### Step 4: Verify Rules

```
ListECRPullThroughCacheRules()
```

### Step 5: Test Container Access

```
CheckContainerAvailability(
    container_uri="docker-hub/biocontainers/bwa:0.7.17"
)
```

### Step 6: Grant HealthOmics Access

For existing repositories:

```
GrantHealthOmicsECRAccess(
    repository_name="my-workflow/bwa"
)
```

## Container URI Mapping

| Original | With Pull-Through |
|----------|------------------|
| `quay.io/biocontainers/bwa:0.7.17` | `ACCOUNT.dkr.ecr.REGION.amazonaws.com/quay/biocontainers/bwa:0.7.17` |
| `docker.io/library/python:3.9` | `ACCOUNT.dkr.ecr.REGION.amazonaws.com/docker-hub/library/python:3.9` |
| `public.ecr.aws/lts/ubuntu:22.04` | `ACCOUNT.dkr.ecr.REGION.amazonaws.com/ecr-public/lts/ubuntu:22.04` |

## Registry Map File

For workflows with many containers, generate a registry map:

```
GenerateContainerRegistryMap(
    workflow_folder="/path/to/workflow",
    output_path="/path/to/registry_map.json"
)
```

Output:
```json
{
    "quay.io/biocontainers/bwa:0.7.17": "ACCOUNT.dkr.ecr.REGION.amazonaws.com/quay/biocontainers/bwa:0.7.17",
    "quay.io/biocontainers/samtools:1.17": "ACCOUNT.dkr.ecr.REGION.amazonaws.com/quay/biocontainers/samtools:1.17"
}
```

## Service Role Permissions

HealthOmics execution role needs ECR access:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage",
                "ecr:BatchCheckLayerAvailability"
            ],
            "Resource": "arn:aws:ecr:REGION:ACCOUNT:repository/*"
        },
        {
            "Effect": "Allow",
            "Action": "ecr:GetAuthorizationToken",
            "Resource": "*"
        }
    ]
}
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Rate limit exceeded | Add Docker Hub credentials |
| Image not found | Verify source URI exists |
| Access denied | Check ECR repository policy |
| Wrong architecture | Ensure x86_64/amd64 images |

## Common Scenarios

### New Workflow
Use ECR URIs directly in workflow:
```wdl
runtime {
    container: "~{ecr_registry}/quay/biocontainers/bwa:0.7.17"
}
```

### Migrating Existing Workflow
1. Generate registry map
2. Use map to update container references
3. Or parameterize ECR prefix

### Access Issues
1. Check pull-through rule exists
2. Verify repository policy
3. Test with `CheckContainerAvailability`
