# Git Integration Guide

Create HealthOmics workflows directly from Git repositories (GitHub, GitLab, Bitbucket).

## When to Use Git Integration

**Use `definitionRepository` when:**
- Workflow source is in GitHub, GitLab, or Bitbucket
- Referencing specific branches, tags, or commits
- Working with nf-core or other public pipelines
- Want automatic sync with source control

**Use traditional packaging when:**
- Local files not in Git
- User provides S3 URI
- Need custom packaging

## Supported Platforms

- GitHub (github.com)
- GitHub Enterprise Server
- GitLab (gitlab.com)
- GitLab Self-Managed
- Bitbucket (bitbucket.org)

## Implementation Steps

### Step 1: Check Existing Connections

```
ListCodeConnections()
```

Look for connection matching required provider.

### Step 2: Create Connection (if needed)

```
CreateCodeConnection(
    provider_type="GitHub",  # or GitLab, Bitbucket
    connection_name="my-github-connection"
)
```

**Important**: User must complete OAuth in AWS Console before connection becomes AVAILABLE.

### Step 3: Parse Repository URL

Extract from URL:
- Owner/organization
- Repository name
- Reference type (branch/tag/commit)

Example: `https://github.com/nf-core/rnaseq` → owner: `nf-core`, repo: `rnaseq`

### Step 4: Create Workflow

```
CreateAHOWorkflow(
    name="nf-core-rnaseq",
    engine="NEXTFLOW",
    definition_repository={
        "connectionArn": "arn:aws:codeconnections:...",
        "repositoryId": "nf-core/rnaseq",
        "sourceReferenceType": "TAG",
        "sourceReference": "3.14.0"
    },
    parameter_template_path="conf/test.config"
)
```

### Step 5: Verify Creation

```
GetAHOWorkflow(workflow_id="...")
# Confirm status is ACTIVE
```

## Repository Reference Types

| Type | Use Case | Example |
|------|----------|---------|
| BRANCH | Development, latest | `main`, `develop` |
| TAG | Releases, stable | `v1.0.0`, `3.14.0` |
| COMMIT | Exact version | `abc123def456...` |

## Examples

### nf-core Pipeline (Tag)

```
CreateAHOWorkflow(
    name="nf-core-rnaseq",
    engine="NEXTFLOW",
    definition_repository={
        "connectionArn": "arn:aws:codeconnections:us-east-1:123456789:connection/abc",
        "repositoryId": "nf-core/rnaseq",
        "sourceReferenceType": "TAG",
        "sourceReference": "3.14.0"
    }
)
```

### Development Branch

```
CreateAHOWorkflow(
    name="my-pipeline-dev",
    engine="WDL",
    definition_repository={
        "connectionArn": "...",
        "repositoryId": "myorg/my-pipeline",
        "sourceReferenceType": "BRANCH",
        "sourceReference": "feature/new-qc"
    }
)
```

### Pinned Commit

```
CreateAHOWorkflow(
    name="my-pipeline-stable",
    engine="WDL",
    definition_repository={
        "connectionArn": "...",
        "repositoryId": "myorg/my-pipeline",
        "sourceReferenceType": "COMMIT",
        "sourceReference": "a1b2c3d4e5f6..."
    }
)
```

## Container Registry Maps

For workflows using public containers, check for registry map file or set up ECR pull-through cache. See [ecr-setup.md](ecr-setup.md).

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Connection PENDING | Complete OAuth in AWS Console |
| Access denied | Verify repository permissions |
| Workflow not found | Check file path in repository |
| Invalid reference | Verify branch/tag/commit exists |

## Required Permissions

IAM policy needs:
- `codeconnections:CreateConnection`
- `codeconnections:GetConnection`
- `codeconnections:ListConnections`
- `codeconnections:UseConnection`
