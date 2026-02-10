# AWS HealthOmics Skill

A Claude Code skill for creating, migrating, running, debugging, and optimizing genomics workflows in AWS HealthOmics.

## Overview

This skill provides comprehensive guidance for working with AWS HealthOmics bioinformatics pipelines, supporting WDL, Nextflow, and CWL workflow languages.

## Features

- **Workflow Development**: Create new genomics pipelines from scratch
- **Migration**: Step-by-step guide for migrating Cromwell/on-premises WDL workflows
- **Git Integration**: Deploy workflows directly from GitHub, GitLab, or Bitbucket
- **Container Setup**: Configure ECR pull-through caches for bioinformatics containers
- **Troubleshooting**: Diagnose and fix failed workflow runs
- **Versioning**: Best practices for updating existing workflows

## Installation

Copy `aws-healthomics.skill` to your Claude Code skills directory.

## Skill Structure

```
aws-healthomics/
├── SKILL.md                    # Main skill instructions
└── references/
    ├── workflow-development.md # Creating new workflows
    ├── migration-guide.md      # Cromwell → HealthOmics migration
    ├── git-integration.md      # GitHub/GitLab deployment
    ├── ecr-setup.md            # Container registry configuration
    ├── troubleshooting.md      # Debugging failed runs
    └── versioning.md           # Workflow version management
```

## Usage

The skill triggers when working with:
- Genomics/bioinformatics workflows
- WDL, Nextflow, or CWL pipelines
- AWS HealthOmics service
- RNA-seq, DNA-seq, variant calling pipelines

## Prerequisites

- AWS account with HealthOmics access
- AWS credentials configured
- S3 bucket for workflow outputs
- ECR repositories (or pull-through cache) for containers

## MCP Server

This skill is designed to work with the [AWS HealthOmics MCP Server](https://awslabs.github.io/mcp/servers/aws-healthomics-mcp-server), which provides tools for:
- Workflow management (create, validate, package)
- Run execution and monitoring
- Failure diagnosis and performance analysis
- Genomics file discovery

## License

Apache-2.0
