---
name: aws-healthomics
description: |
  Create, migrate, run, debug and optimize genomics workflows in AWS HealthOmics.
  Use when working with bioinformatics pipelines supporting WDL, Nextflow, or CWL languages.

  Triggers: (1) Creating new genomics/bioinformatics workflows, (2) Migrating existing
  Cromwell/on-premises WDL pipelines to HealthOmics, (3) Running and monitoring workflow
  executions, (4) Debugging failed workflow runs, (5) Optimizing workflow performance,
  (6) Setting up ECR pull-through caches for containers, (7) Integrating workflows from
  Git repositories (GitHub, GitLab, Bitbucket), (8) Searching for genomics files across
  S3 and HealthOmics stores.

  Keywords: healthomics, WDL, CWL, Nextflow, workflow, genomics, bioinformatics, pipeline,
  RNA-seq, DNA-seq, variant calling, scRNA-seq, omics, sequencing
---

# AWS HealthOmics Workflow Development

## Prerequisites

Before starting, verify:

1. AWS credentials configured (`aws sts get-caller-identity`)
2. HealthOmics service role with S3 and ECR access
3. S3 bucket for workflow outputs in same region
4. ECR repositories accessible to HealthOmics (or set up pull-through cache)

## Quick Reference: Which Guide to Use

| Task | Reference File |
|------|---------------|
| New workflow from scratch | [workflow-development.md](references/workflow-development.md) |
| Migrate existing WDL/Cromwell workflow | [migration-guide.md](references/migration-guide.md) |
| Import from GitHub/GitLab | [git-integration.md](references/git-integration.md) |
| Set up container access | [ecr-setup.md](references/ecr-setup.md) |
| Debug failed runs | [troubleshooting.md](references/troubleshooting.md) |
| Update existing workflow | [versioning.md](references/versioning.md) |

## MCP Server Tools

The AWS HealthOmics MCP server provides these tool categories:

### Workflow Management
- `CreateAHOWorkflow` - Create new workflow from ZIP or S3
- `CreateAHOWorkflowVersion` - Add version to existing workflow
- `LintAHOWorkflowDefinition` - Validate WDL/Nextflow/CWL syntax
- `PackageAHOWorkflow` - Package local files into deployable ZIP
- `ListAHOWorkflows` / `GetAHOWorkflow` - Query workflows

### Execution
- `StartAHORun` - Execute workflow with parameters
- `ListAHORuns` / `GetAHORun` - Monitor run status
- `GetAHORunTask` - Check individual task status

### Troubleshooting
- `DiagnoseAHORunFailure` - Analyze failures with remediation suggestions
- `GetAHORunLog` / `GetAHOEngineLog` / `GetAHOTaskLog` - Access logs
- `AnalyzeAHORunPerformance` - Resource utilization analysis

### File Discovery
- `SearchGenomicsFiles` - Find FASTQ, BAM, VCF files across S3 and stores

## Common Workflows

### Create and Run a New Workflow

```
1. Write workflow definition (WDL 1.1 preferred)
2. LintAHOWorkflowDefinition → validate syntax
3. PackageAHOWorkflow → create deployable ZIP
4. CreateAHOWorkflow → deploy to HealthOmics
5. SearchGenomicsFiles → locate input files
6. StartAHORun → execute with parameters
7. GetAHORun → monitor until completion
```

### Debug a Failed Run

```
1. GetAHORun → check status and error info
2. DiagnoseAHORunFailure → get detailed analysis
3. GetAHOTaskLog → examine specific task failures
4. Fix issues in workflow definition
5. CreateAHOWorkflowVersion → deploy fix
6. StartAHORun → retry
```

### Migrate Cromwell Workflow

```
1. Read references/migration-guide.md
2. Inventory containers → migrate to ECR
3. Add explicit CPU/memory to all tasks
4. Update WDL syntax to 1.0+
5. Move input files to S3
6. Declare all workflow outputs
7. Test with minimal dataset
```

## Language Requirements

### WDL (Preferred)
- Version 1.0 or 1.1 (draft-2 not supported)
- Use `~{}` interpolation (not `${}`)
- Every task needs: `cpu`, `memory`, `container`
- Minimum: 2 vCPU, 4GB memory per task

### Nextflow
- DSL2 syntax required
- Provide `nf-schema.json` for parameters
- Configure resource labels

### CWL
- Version 1.2
- ResourceRequirement for each step

## Container Requirements

All containers must be in private ECR with HealthOmics access:

```
<account>.dkr.ecr.<region>.amazonaws.com/<workflow>/<tool>:<version>
```

Set up ECR pull-through cache to auto-fetch from Docker Hub, Quay.io, or ECR Public. See [ecr-setup.md](references/ecr-setup.md).

## Input/Output Patterns

**Inputs**: All files must be in S3 (same region as HealthOmics)
```
s3://bucket/references/GRCh38/genome.fa
s3://bucket/samples/sample1_R1.fastq.gz
```

**Outputs**: Only declared workflow outputs persist. Intermediate files are deleted.

```wdl
workflow MyWorkflow {
    output {
        File aligned_bam = AlignReads.bam
        File variants = CallVariants.vcf
    }
}
```
