# AWS Resource Explorer Setup

This Terraform project sets up AWS Resource Explorer with indexes across enabled regions in both root and member accounts of an AWS Organization. It also creates necessary IAM roles for managing resources.

## Prerequisites

1. AWS Organization with:
   - Root (management) account
   - One or more member accounts
   - OrganizationAccountAccessRole available in member accounts

2. AWS Identity Center (formerly SSO) configured

3. Required Permissions:
   - Ability to assume TerraformExecutionRole in root account
   - Ability to assume OrganizationAccountAccessRole in member accounts
   - Resource Explorer administrative permissions

## IAM Roles Overview

### Prerequisites Roles
1. **TerraformExecutionRole** (in root account)
   - Used by: Terraform
   - Purpose: Creates initial resources and Lambda function
   - Key Permissions:
     - IAM role and policy management
     - Lambda function management
     - Resource Explorer management
     - Organizations API access

2. **OrganizationAccountAccessRole** (in member accounts)
   - Used by: Lambda function
   - Purpose: Allows cross-account access from management account
   - Must exist before running this project

### Created Roles

1. **P0RoleIamResourceLister** (created in all accounts)
   - Created in: Root and all member accounts
   - Purpose: Allows listing and viewing resources via Resource Explorer
   - Trust Relationship: Google Workspace federation
   - Inline Policy: P0RoleIamResourceListerPolicy

2. **resource-explorer-lambda-role** (in root account only)
   - Created in: Root account
   - Purpose: Execution role for Lambda function
   - Used by: setup-resource-explorer Lambda function
   - Key Permissions:
     - AssumeRole for cross-account access
     - Resource Explorer management
     - CloudWatch Logs
     - IAM role management

## Workflow

1. **Root Account Setup** (via Terraform):
   - Creates P0RoleIamResourceLister with Google federation
   - Creates Resource Explorer indexes in enabled regions
   - Sets up aggregator index in us-west-2
   - Creates and sets default view
   - Creates Lambda function and its execution role

2. **Member Account Setup** (via Lambda):
   For each member account:
   - Assumes OrganizationAccountAccessRole
   - Creates P0RoleIamResourceLister with Google federation
   - Creates Resource Explorer indexes in enabled regions
   - Sets up aggregator index in us-west-2
   - Creates and sets default view

## Project Structure

```
project_root/
├── main.tf                           # Main Terraform config
├── variables.tf                      # Variable definitions
├── provider.tf                       # AWS provider configuration
├── terraform.tfvars                  # Your variable values
├── policies/
│   └── resource_lister_policy.json   # Policy template
└── lambda/
    └── setup_resource_explorer.js    # Lambda function code
```

## Usage

1. Configure `terraform.tfvars`:
```hcl
root_account_id    = "YOUR_ROOT_ACCOUNT_ID"
member_accounts    = ["MEMBER_ACCOUNT_1_ID", "MEMBER_ACCOUNT_2_ID"]
google_audience_id = "YOUR_GOOGLE_AUDIENCE_ID"
```

2. Initialize and apply:
```bash
terraform init
terraform apply
```

3. Monitor Lambda execution:
```bash
aws logs tail /aws/lambda/setup-resource-explorer --follow
```

## Important Notes

- The Lambda function can be rerun safely as it checks for existing resources
- Resource Explorer aggregator setup has a 24-hour cooldown period
- The Lambda can be run with `skipAggregator: true` to test region setup without modifying aggregator settings

## Permissions Flow

```
Your Identity 
  → TerraformExecutionRole (root account)
     → Creates initial resources
     → Creates Lambda function

Lambda Function (root account)
  → OrganizationAccountAccessRole (member accounts)
     → Creates P0RoleIamResourceLister
     → Sets up Resource Explorer
```

## End Result
- Resource Explorer indexes in all enabled regions in all accounts
- Aggregator index in us-west-2 in all accounts
- P0RoleIamResourceLister available in all accounts for Google federation
- Default view configured in us-west-2 for all accounts