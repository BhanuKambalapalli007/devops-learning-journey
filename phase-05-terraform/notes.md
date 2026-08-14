# Terraform overview — Phase 05

## What is Terraform?
Terraform is an Infrastructure as Code (IaC) tool by HashiCorp.
It lets you define cloud infrastructure in human-readable config
files and provision it with a single command.

## Why DevOps engineers use it
- Provision AWS, Azure, GCP resources from code
- Version control your infrastructure
- Reproducible environments — dev, staging, prod identical
- Destroy and rebuild infrastructure in minutes

## Core concepts
- Provider  — AWS, Azure, GCP plugin
- Resource  — EC2 instance, S3 bucket, VPC
- State     — tracks real world vs your config
- Plan      — preview changes before applying
- Apply     — execute the infrastructure changes

## Key commands (Phase 05 preview)
terraform init      # initialise working directory
terraform plan      # show what will change
terraform apply     # create / update infrastructure
terraform destroy   # tear down infrastructure

## Status
⏳ Starting after Phase 04 AWS is complete
