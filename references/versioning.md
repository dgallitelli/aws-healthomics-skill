# Workflow Versioning Guide

## Key Rule

**Always use `CreateAHOWorkflowVersion` to update existing workflows.**

Do NOT create a new workflow for updates. Versioning provides:
- Audit trail of changes
- Easy rollback
- Consistent workflow ID for integrations
- Cost analysis across versions

## When to Version vs Create New

### Use CreateAHOWorkflowVersion

- Bug fixes
- New features/tasks
- Container updates
- Resource adjustments (CPU, memory)
- Parameter changes
- Output modifications
- Performance optimizations
- Fixes after failed runs

### Use CreateAHOWorkflow

- Completely new functionality
- Different analysis type
- User explicitly requests new workflow ID

## Versioning Process

### Step 1: Find Existing Workflow

```
ListAHOWorkflows(name_prefix="my-pipeline")
```

Then get details:
```
GetAHOWorkflow(workflow_id="...")
```

### Step 2: Modify Definition

Edit workflow files locally:
- Fix bugs
- Add features
- Update containers
- Adjust resources

### Step 3: Validate Changes

```
LintAHOWorkflowDefinition(
    workflow_definition_uri="file://main.wdl"
)
```

### Step 4: Package Updates

```
PackageAHOWorkflow(
    workflow_folder="/path/to/workflow"
)
```

### Step 5: Create Version

```
CreateAHOWorkflowVersion(
    workflow_id="existing-workflow-id",
    version="1.1.0",
    definition_zip=<base64_zip>,
    description="Fixed memory issue in alignment task"
)
```

### Step 6: Verify

```
GetAHOWorkflow(workflow_id="...")
```

Confirm new version is ACTIVE.

## Semantic Versioning

Follow MAJOR.MINOR.PATCH:

| Change Type | Version Bump | Example |
|-------------|--------------|---------|
| Breaking changes | MAJOR | 1.0.0 → 2.0.0 |
| New features (compatible) | MINOR | 1.0.0 → 1.1.0 |
| Bug fixes | PATCH | 1.0.0 → 1.0.1 |

### Examples

- `1.0.0` → `1.0.1`: Fixed OOM in alignment task
- `1.0.1` → `1.1.0`: Added QC metrics output
- `1.1.0` → `2.0.0`: Changed input format (breaking)

## Running Specific Versions

```
StartAHORun(
    workflow_id="...",
    workflow_version="1.0.0",  # Specific version
    ...
)
```

Or use latest:
```
StartAHORun(
    workflow_id="...",
    # Omit version for latest
    ...
)
```

## Rollback

To revert to previous version, simply run with old version:

```
StartAHORun(
    workflow_id="...",
    workflow_version="1.0.0",  # Previous working version
    ...
)
```

## Best Practices

1. **Document Changes**: Always include description in version
2. **Test First**: Validate with test data before production
3. **Increment Properly**: Follow semantic versioning
4. **Keep History**: Don't delete old versions
5. **Tag Releases**: Use meaningful version numbers
