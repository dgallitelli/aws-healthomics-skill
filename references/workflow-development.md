# Workflow Development Guide

## Table of Contents
1. [Creating Workflows](#creating-workflows)
2. [Structure Requirements](#structure-requirements)
3. [Resource Specifications](#resource-specifications)
4. [Deploying Workflows](#deploying-workflows)
5. [Running Workflows](#running-workflows)

## Creating Workflows

### Supported Languages
- **WDL 1.1** (preferred) - Workflow Description Language
- **Nextflow DSL2** - Data-driven pipelines
- **CWL 1.2** - Common Workflow Language

### Directory Structure

```
workflow/
├── main.wdl           # Entry point (required at root)
├── tasks/             # Task definitions
│   ├── align.wdl
│   └── call_variants.wdl
├── workflows/         # Sub-workflows
├── parameters.json    # Example inputs
└── README.md          # Documentation
```

### Documentation Requirements
- Comments explaining task purpose
- `meta` blocks with descriptions
- `parameter_meta` for inputs
- `nf-schema.json` for Nextflow

## Structure Requirements

### WDL Best Practices

```wdl
version 1.1

task AlignReads {
    meta {
        description: "Align reads to reference genome using BWA-MEM2"
    }

    parameter_meta {
        reads_1: "Forward reads (FASTQ)"
        reads_2: "Reverse reads (FASTQ)"
        reference: "Reference genome (FASTA)"
    }

    input {
        File reads_1
        File reads_2
        File reference
    }

    command <<<
        set -euo pipefail
        bwa-mem2 mem -t ~{cpu} ~{reference} ~{reads_1} ~{reads_2} > aligned.sam
    >>>

    runtime {
        container: "~{ecr_registry}/bwa-mem2:2.2.1"
        cpu: 8
        memory: "32 GB"
    }

    output {
        File aligned_sam = "aligned.sam"
    }
}
```

### Command Block Guidelines
- Use `set -euo pipefail` for error handling
- Use `~{}` interpolation (not `${}`)
- Quote variables: `"~{filename}"`
- Use `sep()` for arrays: `~{sep=" " files}`

## Resource Specifications

### Required Runtime Attributes

Every task MUST declare:

| Attribute | Minimum | Maximum | Example |
|-----------|---------|---------|---------|
| cpu | 2 | 96 | `cpu: 4` |
| memory | 4 GB | 768 GB | `memory: "16 GB"` |
| container | - | - | `container: "..."` |

### Parallelization

Use scatter for parallel execution:

```wdl
scatter (sample in samples) {
    call AlignReads { input: reads = sample }
}

scatter (interval in intervals) {
    call CallVariants { input: region = interval }
}
```

## Deploying Workflows

### Step 1: Lint Validation
```
LintAHOWorkflowDefinition(
    workflow_definition_uri="file://main.wdl",
    engine="WDL"
)
```

### Step 2: Package Files
```
PackageAHOWorkflow(
    workflow_folder="/path/to/workflow"
)
# Returns base64-encoded ZIP
```

### Step 3: Create Workflow
```
CreateAHOWorkflow(
    name="my-rna-seq-pipeline",
    engine="WDL",
    definition_zip=<base64_zip>,
    main="main.wdl",
    parameter_template="parameters.json"
)
```

### Alternative: S3 Deployment
For large workflows (>10MB):
1. Upload ZIP to S3
2. Use `definition_uri` instead of `definition_zip`

## Running Workflows

### Step 1: Verify Deployment
```
GetAHOWorkflow(workflow_id="...")
# Status should be ACTIVE
```

### Step 2: Prepare Inputs
```
SearchGenomicsFiles(
    query="sample1*.fastq.gz",
    search_paths=["s3://my-bucket/samples/"]
)
```

### Step 3: Start Run
```
StartAHORun(
    workflow_id="...",
    workflow_version="1.0.0",
    name="sample1-analysis",
    parameters={
        "reads_1": "s3://bucket/sample1_R1.fastq.gz",
        "reads_2": "s3://bucket/sample1_R2.fastq.gz",
        "reference": "s3://bucket/refs/GRCh38.fa"
    },
    output_uri="s3://bucket/outputs/",
    role_arn="arn:aws:iam::ACCOUNT:role/HealthOmicsRole"
)
```

### Step 4: Monitor
```
GetAHORun(run_id="...")
# Poll until status is COMPLETED or FAILED
```

## Output Management

Only workflow-level outputs persist:

```wdl
workflow RNAseqPipeline {
    call Align
    call Quantify
    call DifferentialExpression

    output {
        # These files are kept
        File counts = Quantify.counts
        File de_results = DifferentialExpression.results

        # Intermediate BAMs are NOT kept unless declared here
    }
}
```
