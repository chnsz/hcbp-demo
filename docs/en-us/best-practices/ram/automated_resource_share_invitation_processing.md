# Deploy Automated Resource Share Invitation Processing

## Application Scenario

Resource Access Manager (RAM) is a resource sharing service provided by Huawei Cloud, supporting cross-account resource sharing and management to help you achieve unified resource management and access control. Through automated resource share invitation processing, you can batch query pending resource share invitations and automatically accept or reject these invitations, improving the efficiency and convenience of resource sharing management. This best practice introduces how to use Terraform to automatically process resource share invitations, including querying pending invitations and batch accepting or rejecting invitations.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Resource Share Invitations Query Data Source (data.huaweicloud_ram_resource_share_invitations)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/ram_resource_share_invitations)

### Resources

- [Resource Share Accepter Resource (huaweicloud_ram_resource_share_accepter)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ram_resource_share_accepter)

### Resource/Data Source Dependencies

```text
data.huaweicloud_ram_resource_share_invitations
    └── huaweicloud_ram_resource_share_accepter
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../../introductions/prepare_before_deploy.md).

### 2. Query Pending Resource Share Invitations

Add the following script to the TF file (such as main.tf) to query pending resource share invitations:

```hcl
variable "resource_share_ids" {
  description = "List of resource share IDs to query invitations for"
  type        = list(string)
  default     = []
}

# Query pending invitations for specified resource share ID list
data "huaweicloud_ram_resource_share_invitations" "test" {
  resource_share_ids = var.resource_share_ids
  status             = "pending"
}
```

**Parameter Description**:
- **resource_share_ids**: Resource share ID list, assigned by referencing the input variable `resource_share_ids`, used to specify the resource shares to query
- **status**: Invitation status, set to `pending` to query pending invitations

### 3. Process Resource Share Invitations

Add the following script to the TF file (such as main.tf) to process resource share invitations:

```hcl
variable "action" {
  description = "The action to perform on invitations"
  type        = string
  default     = "reject"

  validation {
    condition     = contains(["accept", "reject"], var.action)
    error_message = "The action must be either 'accept' or 'reject'."
  }
}

# Batch process invitations: accept or reject
resource "huaweicloud_ram_resource_share_accepter" "test" {
  count = length(data.huaweicloud_ram_resource_share_invitations.test.resource_share_invitations)

  resource_share_invitation_id = data.huaweicloud_ram_resource_share_invitations.test.resource_share_invitations[count.index].id
  action                       = var.action
}
```

**Parameter Description**:
- **count**: Resource count, dynamically creates resources based on the number of pending invitations queried
- **resource_share_invitation_id**: Resource share invitation ID, assigned by referencing the invitation ID in the data source query results
- **action**: Processing action, assigned by referencing the input variable `action`. Optional values are `accept` (accept) or `reject` (reject), default is `reject`

> Note: Through the `count` parameter, resources can be dynamically created based on the number of invitations queried, achieving batch processing. The processing action can be accept or reject, configured according to actual needs.

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input each time.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Resource share ID list (Required)
resource_share_ids = [
  "resource-share-id-1",
  "resource-share-id-2"
]

# Processing action (Optional, default is reject)
action = "accept"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the variable values in this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command-line parameters: `terraform apply -var="resource_share_ids=['resource-share-id-1','resource-share-id-2']" -var="action=accept"`
2. Environment variables: `export TF_VAR_resource_share_ids='["resource-share-id-1","resource-share-id-2"]'`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command-line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to process resource share invitations:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource processing plan
3. After confirming that the resource plan is correct, run `terraform apply` to start processing resource share invitations
4. Run `terraform show` to view the processed invitations

## Reference Information

- [Huawei Cloud RAM Product Documentation](https://support.huaweicloud.com/ram/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Automated Resource Share Invitation Processing](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ram/automated-resource-share-invitation-processing)
