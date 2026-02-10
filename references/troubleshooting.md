# Troubleshooting Guide

## Workflow Creation Failures

When workflow fails to reach CREATED/ACTIVE status:

### Corrupted Package
**Symptom**: Creation fails immediately
**Solution**: Re-package workflow, verify ZIP integrity

### Multiple Definition Files
**Symptom**: "Multiple workflow definition files at top level"
**Solution**: Keep only one `main.wdl`, `main.nf`, or `main.cwl` at root. Move dependencies to subdirectories.

Correct structure:
```
workflow/
├── main.wdl          # Only one at root
└── tasks/
    ├── align.wdl     # Dependencies in subdirs
    └── qc.wdl
```

### Missing Dependencies
**Symptom**: Import resolution errors
**Solution**:
- Verify all imported files are included in package
- Check import paths match actual file locations

### Invalid Syntax
**Symptom**: Syntax/validation errors
**Solution**: Run linter before deployment:
```
LintAHOWorkflowDefinition(
    workflow_definition_uri="file://main.wdl",
    engine="WDL"
)
```

## Run Failures

### Diagnosing Failures

Always start with:
```
DiagnoseAHORunFailure(run_id="...")
```

Returns:
- Failure type and cause
- Affected tasks
- Remediation suggestions

### Service Errors (5xx)

**Symptom**: Error codes starting with 5
**Cause**: Transient HealthOmics infrastructure issue
**Solution**: Retry the run

```
StartAHORun(
    workflow_id="...",
    parameters=<same_parameters>
)
```

### Customer Errors (4xx)

**Symptom**: Error codes starting with 4
**Cause**: Issue with workflow, inputs, or configuration
**Solution**: Investigate with logs

```
# Get run details
GetAHORun(run_id="...")

# Get specific task logs
GetAHOTaskLog(
    run_id="...",
    task_name="AlignReads"
)

# Get engine logs
GetAHOEngineLog(run_id="...")
```

## Common Runtime Errors

### Container Access Denied

**Symptom**: "CannotPullContainerError" or ECR access denied
**Causes**:
- Container not in ECR
- Missing ECR permissions
- Pull-through cache not configured

**Solutions**:
1. Verify container exists in ECR
2. Check repository policy includes HealthOmics
3. Set up pull-through cache (see [ecr-setup.md](ecr-setup.md))

### File Not Found

**Symptom**: "FileNotFoundException" or S3 access errors
**Causes**:
- Wrong S3 path
- File in different region
- Missing S3 permissions

**Solutions**:
1. Verify S3 URI is correct
2. Ensure file is in same region as HealthOmics
3. Check IAM role has S3 read access

### Out of Memory

**Symptom**: Task killed, OOM errors
**Cause**: Memory allocation insufficient
**Solution**: Increase memory in task runtime:

```wdl
runtime {
    memory: "32 GB"  # Increase from 16 GB
}
```

Then create new workflow version.

### Disk Space Exceeded

**Symptom**: "No space left on device"
**Cause**: Task output exceeds allocated storage
**Solutions**:
1. Increase storage in run configuration
2. Stream outputs directly to S3
3. Clean up intermediate files in task

### Timeout

**Symptom**: Task or run timeout
**Cause**: Task taking longer than allowed
**Solutions**:
1. Optimize task (more CPUs, better algorithm)
2. Split into smaller chunks (scatter)
3. Increase timeout in configuration

## Debugging Workflow

### Step-by-Step Process

1. **Get Run Status**
   ```
   GetAHORun(run_id="...")
   ```
   Check: status, error message, failed task

2. **Diagnose Failure**
   ```
   DiagnoseAHORunFailure(run_id="...")
   ```
   Get: root cause, remediation steps

3. **Check Task Logs**
   ```
   GetAHOTaskLog(run_id="...", task_name="FailedTask")
   ```
   Look for: error messages, stack traces

4. **Check Engine Logs**
   ```
   GetAHOEngineLog(run_id="...")
   ```
   Look for: workflow-level errors, resource issues

5. **Fix and Retry**
   - Update workflow definition
   - Create new version
   - Start new run

### Log Types

| Log | Contains | Use When |
|-----|----------|----------|
| Run Log | Overall run status | Initial investigation |
| Task Log | Task stdout/stderr | Task-specific errors |
| Engine Log | Workflow engine output | Syntax/orchestration issues |
| Manifest Log | File manifest | Input/output problems |

## Performance Issues

For slow or inefficient runs:

```
AnalyzeAHORunPerformance(run_id="...")
```

Returns:
- Resource utilization per task
- Bottleneck identification
- Optimization recommendations

### Common Optimizations

| Issue | Solution |
|-------|----------|
| Under-utilized CPU | Reduce cpu allocation |
| Memory under-used | Reduce memory allocation |
| Long scatter time | Increase parallelism |
| Slow I/O | Use DYNAMIC storage |
