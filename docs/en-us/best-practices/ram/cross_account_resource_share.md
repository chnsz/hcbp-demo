# Deploy Cross-Account Resource Share

## Application Scenario

Resource Access Manager (RAM) is a resource sharing service provided by Huawei Cloud, supporting cross-account resource sharing and management to help you achieve unified resource management and access control. Through cross-account resource sharing, network resources such as VPCs, subnets, and security groups, as well as computing and storage resources such as ECS and RDS, can be shared with other accounts or organizations, achieving unified resource management and access control. This best practice introduces how to use Terraform to automatically deploy cross-account resource sharing, including resource share instance creation, principal configuration, resource URN configuration, and permission configuration.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Resources

- [Resource Share Resource (huaweicloud_ram_resource_share)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ram_resource_share)

### Resource/Data Source Dependencies

```text
huaweicloud_ram_resource_share
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../../introductions/prepare_before_deploy.md).

### 2. Create Resource Share

Add the following script to the TF file (such as main.tf) to create a resource share:

```hcl
variable "resource_share_name" {
  description = "The name of the resource share"
  type        = string
}

variable "description" {
  description = "The description of the resource share"
  type        = string
  default     = ""
}

variable "principals" {
  description = "The list of one or more principals (account IDs or organization IDs) to share resources with"
  type        = list(string)
}

variable "resource_urns" {
  description = "The list of URNs of one or more resources to be shared. If not specified, URNs will be automatically generated from created resources (VPC, subnet, security group)"
  type        = set(string)
  default     = []
}

variable "permission_ids" {
  description = "The list of RAM permissions associated with the resource share"
  type        = list(string)
  default     = []
}

variable "allow_external_principals" {
  description = "Whether resources can be shared with any accounts outside the organization"
  type        = bool
  default     = false
}

# Create resource share resource in the specified region (defaults to the region specified in the current provider block when region parameter is omitted)
resource "huaweicloud_ram_resource_share" "test" {
  name                      = var.resource_share_name
  description               = var.description
  principals                = var.principals
  resource_urns             = var.resource_urns
  permission_ids            = var.permission_ids
  allow_external_principals = var.allow_external_principals
}
```

**Parameter Description**:
- **name**: Resource share name, assigned by referencing the input variable `resource_share_name`
- **description**: Resource share description, assigned by referencing the input variable `description`, optional parameter
- **principals**: Principal list, assigned by referencing the input variable `principals`, used to specify the list of account IDs or organization IDs to share resources with
- **resource_urns**: Resource URN list, assigned by referencing the input variable `resource_urns`, used to specify the list of resource URNs to be shared, optional parameter
- **permission_ids**: Permission ID list, assigned by referencing the input variable `permission_ids`, used to specify the list of RAM permissions associated with the resource share, optional parameter
- **allow_external_principals**: Whether resources can be shared with any accounts outside the organization, assigned by referencing the input variable `allow_external_principals`, default is `false`

> Note: Resource URN is the unique identifier of a resource, in the format `resource_type:region:account_id:resource_type:resource_id`. If resource URNs are not specified, the system will automatically generate them based on created resources (VPC, subnet, security group, etc.). Principals can be account IDs or organization IDs, used to specify the recipients of shared resources.

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input each time.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Resource share basic information (Required)
resource_share_name = "cross-account-vpc-share"
description         = "Share VPC resources with other accounts in the organization"

# Principals: Account IDs or Organization IDs to share resources with (Required)
# Should be replaced with real account IDs
principals = [
  "01234567890123456789012345678901",
  "98765432109876543210987654321098"
]

# The list of URNs of one or more resources to be shared (Optional)
# Should be replaced with real resource URNs
resource_urns = [
  "vpc:cn-north-4:8f06724e5c6f41f59d3e2f3ad897bb4d:subnet:5de72eeb-7977-4602-8186-8766982d9bcc"
]

# The list of RAM permissions associated with the resource share (Optional)
# Should be replaced with real permission IDs
permission_ids = [
  "f5153698-ca8b-4b3c-a839-13ff71f67885"
]
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the variable values in this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command-line parameters: `terraform apply -var="resource_share_name=cross-account-vpc-share" -var="principals=['01234567890123456789012345678901']"`
2. Environment variables: `export TF_VAR_resource_share_name=cross-account-vpc-share`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command-line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the resource share
4. Run `terraform show` to view the created resource share

## Reference Information

- [Huawei Cloud RAM Product Documentation](https://support.huaweicloud.com/ram/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Cross-Account Resource Share](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ram/cross-account-resource-share)
