---
title: "Infrastructure templates: provisioning AWS resources from Backstage"
tags: [platform-engineering, backstage, terraform, aws, ecs, iam, account-vending]
series_part: 7
toc: true
description: "A Software Template that provisions a new service's AWS infrastructure — ECR repo, ECS service, IAM roles, SSM parameters — and a separate template that vends a full AWS account baseline from a form."
---

This is part 7 of the [Building a Golden Path with Backstage series](/series/#platform-engineering-series).

The companion repository is at [github.com/mguarinos/backstage-golden-path](https://github.com/mguarinos/backstage-golden-path). Terraform modules live in `terraform/modules/`.

---

## Two kinds of infrastructure templates

This post covers two distinct templates:

**`new-service-infra`**: called by the `new-python-service` and `new-node-service` templates as a step. Provisions the AWS resources a single service needs: ECR repository, ECS cluster and service, IAM task role with least-privilege policy, and SSM parameters for configuration. Scoped to one service in one environment.

**`new-aws-account`**: provisions a full account baseline for a new team or environment. VPC with public and private subnets across two availability zones, IAM roles for developers and CI/CD, CloudTrail, an ECR registry, and the OIDC provider for GitHub Actions. A team gets a fully wired account from a form.

Both templates dispatch a GitHub Actions workflow in a central `platform-infra` repository that runs `terraform apply`. The template never runs Terraform directly — the workflow owns the state and the blast radius.

---

## The `service-infra` Terraform module

```hcl
# terraform/modules/service-infra/variables.tf
variable "service_name"  { type = string }
variable "environment"   { type = string }
variable "aws_region"    { type = string  default = "eu-west-1" }
variable "container_port" { type = number default = 8000 }
variable "cpu"            { type = number default = 256 }
variable "memory"         { type = number default = 512 }
```

**What it creates:**

```hcl
# ECR repository
resource "aws_ecr_repository" "service" {
  name                 = "${var.service_name}-${var.environment}"
  image_tag_mutability = "IMMUTABLE"

  image_scanning_configuration { scan_on_push = true }
  encryption_configuration     { encryption_type = "AES256" }
}

# ECS task IAM role
resource "aws_iam_role" "task" {
  name = "${var.service_name}-${var.environment}-task"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

# Least-privilege: SSM read for this service's params only
resource "aws_iam_role_policy" "ssm_read" {
  name   = "ssm-read"
  role   = aws_iam_role.task.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["ssm:GetParameter", "ssm:GetParametersByPath"]
      Resource = "arn:aws:ssm:${var.aws_region}:*:parameter/${var.service_name}/${var.environment}/*"
    }]
  })
}

# SSM parameters (empty, populated by deployment pipeline)
resource "aws_ssm_parameter" "database_url" {
  name  = "/${var.service_name}/${var.environment}/DATABASE_URL"
  type  = "SecureString"
  value = "placeholder"

  lifecycle { ignore_changes = [value] }
}
```

The `ignore_changes = [value]` on SSM parameters is intentional — Terraform creates the parameter, the deployment pipeline populates it, and subsequent Terraform runs do not overwrite the real value.

**Outputs:**
```hcl
output "ecr_repository_url" { value = aws_ecr_repository.service.repository_url }
output "task_role_arn"       { value = aws_iam_role.task.arn }
output "execution_role_arn"  { value = aws_iam_role.execution.arn }
```

---

## Triggering Terraform from a Backstage template

The `new-python-service` template dispatches a GitHub Actions workflow after creating the repo:

```yaml
- id: provision-infra
  name: Provision AWS infrastructure
  action: github:actions:dispatch
  input:
    repoUrl: github.com?owner=${{ parameters.github_org }}&repo=platform-infra
    workflowId: provision-service.yml
    branchOrTagName: main
    workflowInputs:
      service_name: ${{ parameters.name }}
      environment:  dev
      aws_region:   ${{ parameters.aws_region }}
```

The `provision-service.yml` workflow in `platform-infra` uses OIDC to assume a Terraform execution role and runs:

```yaml
- name: Terraform apply
  run: |
    terraform -chdir=services/${{ inputs.service_name }} init
    terraform -chdir=services/${{ inputs.service_name }} apply -auto-approve
  env:
    TF_VAR_service_name: ${{ inputs.service_name }}
    TF_VAR_environment:  ${{ inputs.environment }}
```

The workflow creates the Terraform workspace directory if it does not exist, writes `main.tf` calling the `service-infra` module, and commits it. Subsequent runs are idempotent.

---

## The `account-baseline` module

```hcl
# terraform/modules/account-baseline/variables.tf
variable "account_name" { type = string }
variable "environment"  { type = string }
variable "vpc_cidr"     { type = string  default = "10.0.0.0/16" }
variable "aws_region"   { type = string  default = "eu-west-1" }
```

**What it creates:**

- VPC with 2 public and 2 private subnets across two AZs
- Internet Gateway and NAT Gateway (one per AZ for HA)
- IAM roles: `developer` (read-only + ECR pull), `cicd` (deploy to ECS, push to ECR), `readonly`
- OIDC provider for GitHub Actions (`token.actions.githubusercontent.com`) with trust policy scoped to the org
- CloudTrail trail writing to S3
- ECR registry (account-level)

**The GitHub Actions OIDC role** is the critical piece — it means every new account is immediately usable from GitHub Actions without storing any credentials:

```hcl
resource "aws_iam_role" "github_actions" {
  name = "github-actions-cicd"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Federated = aws_iam_openid_connect_provider.github.arn }
      Action    = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringLike = {
          "token.actions.githubusercontent.com:sub" = "repo:${var.github_org}/*:*"
        }
      }
    }]
  })
}
```

---

## The `new-aws-account` Backstage template

```yaml
spec:
  parameters:
    - title: Account details
      properties:
        account_name:
          title: Account name
          type: string
          description: Used as a prefix for all resources (e.g. "payments-prod")
        team:
          title: Owning team
          type: string
          ui:field: OwnerPicker
        environment:
          title: Environment
          type: string
          enum: [dev, staging, prod]
        vpc_cidr:
          title: VPC CIDR
          type: string
          default: "10.0.0.0/16"
        aws_org_parent_id:
          title: AWS Organisations parent OU ID
          type: string
          description: "ou-xxxx-xxxxxxxx — the OU this account will be created under"
```

The template dispatches a `provision-account.yml` workflow which:
1. Creates the AWS account via `aws organizations create-account`
2. Assumes the `OrganizationAccountAccessRole` in the new account
3. Runs `terraform apply` for the `account-baseline` module
4. Registers the account in the Backstage catalog as a `Resource` kind

After about ten minutes, the requesting team has a fully wired AWS account — VPC, IAM roles, OIDC configured, CloudTrail running — and it appears in the Backstage catalog linked to their group.

> 📷 **Screenshot** — The `new-aws-account` template form in Backstage, with account name, team, environment, and VPC CIDR fields.

> 📷 **Screenshot** — The GitHub Actions workflow run triggered by the template — showing the Terraform plan output and apply steps completing.

---

## Cost tagging by default

Every resource created by both modules is tagged at the module level:

```hcl
locals {
  tags = {
    Service     = var.service_name
    Environment = var.environment
    ManagedBy   = "terraform"
    Team        = var.team
  }
}
```

These tags flow into AWS Cost Explorer automatically. A team can filter by `Service=payments-service` to see exactly what that service costs — without any manual setup.

Part 8 shows how to make APIs first-class entities in the catalog and build the dependency graph that makes the relationship between services visible.
