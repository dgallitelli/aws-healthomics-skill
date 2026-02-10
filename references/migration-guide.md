# WDL Migration Guide

Migrate on-premises or Cromwell-based WDL workflows to AWS HealthOmics.

## Table of Contents
1. [Migration Overview](#migration-overview)
2. [Phase 1: Container Migration](#phase-1-container-migration)
3. [Phase 2: Runtime Standardization](#phase-2-runtime-standardization)
4. [Phase 3: WDL Syntax Updates](#phase-3-wdl-syntax-updates)
5. [Phase 4: Data Migration](#phase-4-data-migration)
6. [Phase 5: Output Declarations](#phase-5-output-declarations)
7. [Phase 6: Validation](#phase-6-validation)

## Migration Overview

### HealthOmics Requirements
- Containers in private ECR with permissions
- All inputs in S3 (same region)
- Explicit CPU/memory per task
- WDL 1.0+ syntax (no draft-2)
- Declared workflow outputs only persist

### Checklist
- [ ] All containers migrated to ECR
- [ ] Every task has cpu, memory, container
- [ ] WDL syntax updated to 1.0+
- [ ] Input files moved to S3
- [ ] All outputs declared
- [ ] Test run successful

## Phase 1: Container Migration

### 1.1 Inventory Containers
Scan all WDL files for `docker:` and `container:` attributes:

```bash
grep -r "docker:" *.wdl
grep -r "container:" *.wdl
```

### 1.2 Create ECR Repositories
For each container, create ECR repo with HealthOmics access:

```bash
aws ecr create-repository --repository-name workflow-name/tool-name
```

### 1.3 Pull and Push Images

```bash
# Pull from source
docker pull quay.io/biocontainers/bwa:0.7.17

# Tag for ECR
docker tag quay.io/biocontainers/bwa:0.7.17 \
    ACCOUNT.dkr.ecr.REGION.amazonaws.com/workflow/bwa:0.7.17

# Push to ECR
docker push ACCOUNT.dkr.ecr.REGION.amazonaws.com/workflow/bwa:0.7.17
```

### 1.4 Update WDL References

Before:
```wdl
runtime {
    docker: "quay.io/biocontainers/bwa:0.7.17"
}
```

After:
```wdl
runtime {
    container: "~{ecr_registry}/workflow/bwa:0.7.17"
    cpu: 4
    memory: "8 GB"
}
```

## Phase 2: Runtime Standardization

### Resource Boundaries
| Attribute | Minimum | Maximum |
|-----------|---------|---------|
| cpu | 2 | 96 |
| memory | 4 GB | 768 GB |

### Add to Every Task

```wdl
task MyTask {
    runtime {
        container: "..."
        cpu: 4           # Required
        memory: "16 GB"  # Required
    }
}
```

### Validate Coverage
Ensure 100% of tasks have runtime attributes.

## Phase 3: WDL Syntax Updates

### Version Declaration
Add at top of every WDL file:

```wdl
version 1.1
```

### Interpolation Syntax

Before (draft-2):
```wdl
command {
    bwa mem ${reference} ${reads}
}
```

After (1.0+):
```wdl
command <<<
    bwa mem ~{reference} ~{reads}
>>>
```

### Command Block Format

Before:
```wdl
command {
    ...
}
```

After:
```wdl
command <<<
    set -euo pipefail
    ...
>>>
```

### Import Paths
Ensure all imports resolve correctly:

```wdl
import "tasks/align.wdl" as Align
import "tasks/qc.wdl" as QC
```

## Phase 4: Data Migration

### 1. Inventory Files
Identify all File inputs and hardcoded paths.

### 2. Design S3 Structure

```
s3://my-genomics-bucket/
├── references/
│   └── GRCh38/
│       ├── genome.fa
│       ├── genome.fa.fai
│       └── genome.dict
├── annotations/
│   └── gencode.v38.gtf
└── samples/
    ├── sample1/
    │   ├── sample1_R1.fastq.gz
    │   └── sample1_R2.fastq.gz
    └── sample2/
        └── ...
```

### 3. Upload Files

```bash
aws s3 cp ./references/ s3://bucket/references/ --recursive
aws s3 cp ./samples/ s3://bucket/samples/ --recursive
```

### 4. Update Inputs JSON

Before:
```json
{
    "workflow.reference": "/data/refs/genome.fa"
}
```

After:
```json
{
    "workflow.reference": "s3://bucket/references/GRCh38/genome.fa"
}
```

## Phase 5: Output Declarations

### Critical Rule
Only declared workflow outputs persist. Undeclared intermediate files are deleted.

### Audit Outputs
Review what files are needed from each task.

### Declare at Workflow Level

```wdl
workflow MyPipeline {
    call Align
    call CallVariants
    call Annotate

    output {
        File aligned_bam = Align.bam
        File aligned_bai = Align.bai
        File variants = CallVariants.vcf
        File annotated = Annotate.vcf
        # BAM index from CallVariants NOT kept unless listed
    }
}
```

### Task Output Validation
Ensure glob patterns are correct:

```wdl
output {
    Array[File] logs = glob("*.log")
    File bam = "output.bam"
}
```

## Phase 6: Validation

### 1. Lint Check
```
LintAHOWorkflowDefinition(
    workflow_definition_uri="file://main.wdl"
)
```

### 2. Create Test Inputs
Use minimal data (chr22 subset, few samples):

```json
{
    "workflow.reference": "s3://bucket/refs/chr22.fa",
    "workflow.samples": ["s3://bucket/test/small_sample.fastq.gz"],
    "workflow.intervals": "s3://bucket/refs/chr22.intervals"
}
```

### 3. Test Run
Target < 2 hours with test data:

```
StartAHORun(
    workflow_id="...",
    parameters=<test_inputs>,
    storage_type="DYNAMIC"
)
```

### 4. Diagnose Failures
```
DiagnoseAHORunFailure(run_id="...")
```

### 5. Full-Scale Test
After test passes, run with production data.

### 6. Performance Analysis
```
AnalyzeAHORunPerformance(run_id="...")
```

## Common Issues

| Issue | Solution |
|-------|----------|
| Container access denied | Add ECR policy for HealthOmics |
| File not found | Verify S3 path and permissions |
| Memory exceeded | Increase memory in runtime |
| Syntax error | Run linter, check version 1.0+ |
| Missing output | Add to workflow output block |
