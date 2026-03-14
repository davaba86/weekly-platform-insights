# Terraform Multi-Account AWS Deployments Using GitHub Actions OIDC

## TL;DR

This document explains how to setup a GitHub Actions workflow to deploy Terraform code to multiple AWS accounts using OpenID Connect (OIDC) for authentication.

We're also strictly deploying to accounts where changes have been made to the Terraform code which in a large multi-account environment can save a lot of time and reduce the blast radius of any potential issues.

## GHA to AWS Authentication via OIDC

We want to avoid using long-lived AWS credentials in GitHub Actions, and instead use OIDC to assume roles in the target AWS accounts. This is a more secure approach and eliminates the need to manage secrets in GitHub.

## Terraform State Management

S3 is both used to store the Terraform state file and for state locking to prevent concurrent modifications. In the past it was common to use DynamoDB for state locking, but S3 now supports this natively with the `use_lockfile` option.

## Repository Structure

For the example to work you need to have a repository structure similar to the following, with Terraform code for each account in separate directories. The `platform-accounts.json` file contains the necessary information for the workflow to discover which accounts to deploy to and how to authenticate.

```tree
└── terraform
    ├── modules
    ├── platform
    │   ├── audit
    │   │   └── main.tf
    │   ├── log-archive
    │   │   └── main.tf
    │   ├── management
    │   │   └── main.tf
    │   ├── networking
    │   │   └── main.tf
    │   ├── sandbox
    │   │   └── main.tf
    │   ├── security-tooling
    │   │   └── main.tf
    │   └── shared-services
    │       └── main.tf
    └── workloads
        ├── customer-acceptance
        │   └── main.tf
        ├── customer-development
        │   └── main.tf
        └── customer-production
            └── main.tf
```

## JSON Structure

Below is an example of the `platform-accounts.json` file which contains the necessary information for the workflow to discover which accounts to deploy to and how to authenticate. You can add as many accounts as needed following the same structure.

```json filename="terraform/platform-accounts.json"
[
  {
    "account_name": "audit",
    "working_directory": "terraform/platform/audit",
    "role_arn": "arn:aws:iam::111111111111:role/github-terraform-deploy-1111",
    "aws_region": "eu-west-1"
  },
  {
    "account_name": "log-archive",
    "working_directory": "terraform/platform/log-archive",
    "role_arn": "arn:aws:iam::222222222222:role/github-terraform-deploy-2222",
    "aws_region": "eu-west-1"
  }
]

```

## GitHub Actions Workflow

In the first job we're discovering which accounts have changes to deploy by comparing the changed files in the commit or pull request. This allows us to build a dynamic matrix of accounts to deploy to which will be used for the Terraform deployment job.

Once the matrix is built, the second job will run for each account in the matrix. It will use the `aws-actions/configure-aws-credentials` action to assume the specified role in the target AWS account using OIDC, and then run the Terraform commands to deploy the infrastructure.

```yaml filename=".github/workflows/deploy-platform.yml"
---
name: Deploy Platform

on:
  push:
    branches:
      - main
    paths:
      - "terraform/platform/**"
      - "terraform/modules/**"

  pull_request:
    paths:
      - "terraform/platform/**"
      - "terraform/modules/**"

env:
  WORKING_DIR: ./terraform

jobs:
  discover:
    name: Discover changed account(s)
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.discover.outputs.matrix }}

    steps:
      - name: Checkout
        uses: actions/checkout@v6
        with:
          fetch-depth: 0

      - name: Build matrix
        id: discover
        shell: bash
        run: |
          set -euo pipefail

          if [[ "${{ github.event_name }}" == "pull_request" ]]; then
            git diff --name-only "${{ github.event.pull_request.base.sha }}" "${{ github.event.pull_request.head.sha }}" > changed_files.txt
          else
            git diff --name-only "${{ github.event.before }}" "${{ github.sha }}" > changed_files.txt
          fi

          python3 <<'PY'
          import json
          import os
          from pathlib import Path

          changed_files = Path("changed_files.txt").read_text().splitlines()
          catalog = json.loads(Path("terraform/platform-accounts.json").read_text())

          changed_accounts = set()

          for path in changed_files:
              parts = path.split("/")
              if len(parts) >= 3 and parts[0] == "terraform" and parts[1] == "platform":
                  changed_accounts.add(parts[2])

          selected = [entry for entry in catalog if entry["account_name"] in changed_accounts]
          output = {"include": selected}

          with open(os.environ["GITHUB_OUTPUT"], "a") as f:
              f.write(f"matrix={json.dumps(output, separators=(',', ':'))}\n")
          PY

      - name: Show matrix
        run: echo '${{ steps.discover.outputs.matrix }}'

  terraform:
    name: Deploy to ${{ matrix.account_name }}
    needs: discover
    if: needs.discover.outputs.matrix != '{"include":[]}'
    runs-on: ubuntu-latest

    strategy:
      matrix: ${{ fromJSON(needs.discover.outputs.matrix) }}

    permissions:
      contents: read   # Required to checkout code
      id-token: write  # Required for OIDC authentication

    defaults:
      run:
        working-directory: ${{ matrix.working_directory }}

    steps:
      - name: Checkout
        uses: actions/checkout@v6

      - name: Show target
        run: |
          echo "Account: ${{ matrix.account_name }}"
          echo "Dir: ${{ matrix.working_directory }}"

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v4

      - name: Configure AWS credentials from GitHub OIDC
        uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: ${{ matrix.role_arn }}
          aws-region: ${{ matrix.aws_region }}
          role-session-name: gha-terraform-${{ matrix.account_name }}

      - name: Show caller identity
        run: aws sts get-caller-identity

      - name: Terraform fmt
        run: terraform fmt -check -recursive

      - name: Terraform init
        run: terraform init -input=false

      - name: Terraform validate
        run: terraform validate

      - name: Terraform plan
        run: terraform plan -input=false -no-color

      - name: Terraform Apply
        run: terraform apply -auto-approve
```
