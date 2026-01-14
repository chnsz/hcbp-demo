# Deploy Group Policies Associate

## Application Scenario

Identity and Access Management (IAM) is a basic identity authentication and access management service provided by Huawei Cloud, providing core functions such as identity management, permission management, and access control for Huawei Cloud users. By associating policies with user groups, you can uniformly grant permissions to users in the group, achieving permission management based on user groups. This best practice will introduce how to use Terraform to automatically deploy IAM user group and policy associations, including querying IAM policies, creating user groups, and associating policies with user groups.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [IAM Policies Data Source (huaweicloud_identityv5_policies)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/identityv5_policies)

### Resources

- [IAM User Group Resource (huaweicloud_identityv5_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identityv5_group)
- [IAM Policy Group Attach Resource (huaweicloud_identityv5_policy_group_attach)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identityv5_policy_group_attach)

### Resource/Data Source Dependencies

```text
data.huaweicloud_identityv5_policies
    └── huaweicloud_identityv5_policy_group_attach

huaweicloud_identityv5_group
    └── huaweicloud_identityv5_policy_group_attach
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Query IAM Policies Data Source

Add the following script to the TF file (e.g., main.tf) to query IAM policy information:

```hcl
variable "policy_type" {
  description = "The type of the policy"
  type        = string
  default     = "system"
}

variable "policy_names" {
  description = "The name list of policies to be associated with the user group"
  type        = list(string)
}

# Get all IAM policy information in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block), used for associating with user groups
data "huaweicloud_identityv5_policies" "test" {
  policy_type = var.policy_type
}

# Filter policy list by policy name
locals {
  filtered_policies = [for policy in data.huaweicloud_identityv5_policies.test.policies : policy if contains(var.policy_names, policy.policy_name)]
}
```

**Parameter Description**:
- **policy_type**: Policy type, assigned by referencing input variable policy_type, default value is "system" (system policy)
- **policy_names**: Policy name list, assigned by referencing input variable policy_names, used to filter policies that need to be associated
- **locals.filtered_policies**: Filtered policy list based on policy names, used for subsequent association with user groups

### 3. Create IAM User Group Resource

Add the following script to the TF file (e.g., main.tf) to create an IAM user group:

```hcl
variable "group_name" {
  description = "The name of the user group"
  type        = string
}

variable "group_description" {
  description = "The description of the user group"
  type        = string
  default     = ""
}

# Create IAM user group resource in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block)
resource "huaweicloud_identityv5_group" "test" {
  group_name  = var.group_name
  description = var.group_description
}
```

**Parameter Description**:
- **group_name**: User group name, assigned by referencing input variable group_name
- **description**: User group description, assigned by referencing input variable group_description, optional parameter, default value is empty string

### 4. Create IAM Policy Group Attach Resource

Add the following script to the TF file (e.g., main.tf) to associate policies with user groups:

```hcl
# Create IAM policy group attach resource in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block)
resource "huaweicloud_identityv5_policy_group_attach" "test" {
  count = length(local.filtered_policies)

  policy_id = try(local.filtered_policies[count.index].policy_id, null)
  group_id  = huaweicloud_identityv5_group.test.id
}
```

**Parameter Description**:
- **count**: Number of resource creations, dynamically creates attach resources based on the length of the filtered policy list
- **policy_id**: Policy ID, assigned by referencing the filtered policy list
- **group_id**: User group ID, assigned by referencing the user group resource

> Note: The policy group attach resource uses the count parameter to dynamically create multiple attach resources based on the filtered policy list. Ensure that the policies in the policy name list exist in IAM, otherwise the filtered policy list may be empty, resulting in failure to create associations.

### 5. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# IAM User Group Configuration
group_name        = "tf_test_group"
group_description = "Test user group"

# IAM Policy Configuration
policy_type  = "system"
policy_names = [
  "ModelArtsFullAccessPolicy",
  "SCMReadOnlyPolicy"
]
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs, especially:
   - `group_name` needs to be set to user group name
   - `group_description` can be set to user group description information, optional parameter
   - `policy_type` can be set to "system" (system policy) or other policy types
   - `policy_names` needs to be set to the list of policy names to be associated, ensuring these policies exist in IAM
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="group_name=my-group" -var='policy_names=["Policy1","Policy2"]'`
2. Environment variables: `export TF_VAR_group_name=my-group` and `export TF_VAR_policy_names='["Policy1","Policy2"]'`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values. Ensure that the policies in the policy name list exist in IAM, otherwise the filtered policy list may be empty, resulting in failure to create associations.

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create IAM user group and policy associations:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating IAM user group and policy associations
4. Run `terraform show` to view the details of the created IAM user group and policy associations

> Note: After the policy group attach is created, users in the user group will receive the permissions granted by the associated policies. Ensure that policy names are correct, otherwise the filtered policy list may be empty, resulting in failure to create associations. It is recommended to query the available policy list in IAM before creating associations to confirm that policy names are correct.

## Reference Information

- [Huawei Cloud IAM Product Documentation](https://support.huaweicloud.com/iam/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Group Policies Associate](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/iam/v5/group-policies-associate)
