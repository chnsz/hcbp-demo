# Deploy Users Authorized Through Group

## Application Scenario

Identity and Access Management (IAM) is a fundamental identity authentication and access management service provided by Huawei Cloud, achieving fine-grained permission control through flexible combinations of users, user groups, roles, and policies. Authorizing users through user groups is a common permission management approach that can simplify permission management processes and improve management efficiency.

This best practice will introduce how to use Terraform to automatically deploy IAM roles, user groups, and users, and authorize users through user groups. Through this approach, you can implement group-based permission management. When you need to grant the same permissions to multiple users, you only need to add users to the corresponding user group, and they will automatically inherit the user group's permissions without needing to configure permissions for each user individually.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [IAM Role Query Data Source (data.huaweicloud_identity_role)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/identity_role)
- [IAM Project Query Data Source (data.huaweicloud_identity_projects)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/identity_projects)

### Resources

- [IAM Role Resource (huaweicloud_identity_role)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identity_role)
- [IAM User Group Resource (huaweicloud_identity_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identity_group)
- [IAM Role Assignment Resource (huaweicloud_identity_role_assignment)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identity_role_assignment)
- [Random Password Resource (random_password)](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [IAM User Resource (huaweicloud_identity_user)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identity_user)
- [IAM User Group Membership Resource (huaweicloud_identity_group_membership)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/identity_group_membership)

### Resource/Data Source Dependencies

```
data.huaweicloud_identity_role.test
    └── huaweicloud_identity_role_assignment.test

huaweicloud_identity_role.test
    └── huaweicloud_identity_role_assignment.test

data.huaweicloud_identity_projects.test
    └── huaweicloud_identity_role_assignment.test

huaweicloud_identity_group.test
    ├── huaweicloud_identity_role_assignment.test
    └── huaweicloud_identity_group_membership.test

random_password.test
    └── huaweicloud_identity_user.test

huaweicloud_identity_user.test
    └── huaweicloud_identity_group_membership.test
```

## Implementation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) for configuration introduction.

### 2. Query IAM Role Information Through Data Source

Add the following script to the TF file (such as main.tf) to instruct Terraform to perform a data source query, the query results are used to create IAM role assignment resources:

```hcl
variable "role_id" {
  description = "The ID of the IAM role"
  type        = string
  default     = ""
  nullable    = false
}

variable "role_policy" {
  description = "The policy of the IAM role"
  type        = string
  default     = ""
  nullable    = false
}

variable "role_name" {
  description = "The name of the IAM role"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = !(var.role_name == "" && var.role_id == "")
    error_message = "The role_name must be provided when role_id is not provided."
  }
}

# Query IAM role information that meets the conditions in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block), used to create IAM role assignment resources
data "huaweicloud_identity_role" "test" {
  count = var.role_id == "" && var.role_policy == "" ? 1 : 0

  name = var.role_name
}
```

**Parameter Description**:
- **count**: The number of data sources to create, used to control whether to execute the IAM role query data source. The data source is only created (i.e., IAM role query is executed) when `var.role_id` is empty and `var.role_policy` is empty
- **name**: The name of the IAM role, assigned by referencing the input variable `role_name`

### 3. Create IAM Role Resource

Add the following script to the TF file to instruct Terraform to create IAM role resources:

```hcl
variable "role_type" {
  description = "The type of the IAM role"
  type        = string
  default     = "XA"
}

variable "role_description" {
  description = "The description of the IAM role"
  type        = string
  default     = ""

  validation {
    condition     = !(var.role_description == "" && var.role_policy != "")
    error_message = "The role_description must be provided when role_policy is provided."
  }
}

# Create IAM role resource in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block)
resource "huaweicloud_identity_role" "test" {
  count = var.role_id == "" && var.role_policy != "" ? 1 : 0

  name        = var.role_name
  type        = var.role_type
  description = var.role_description
  policy      = var.role_policy
}
```

**Parameter Description**:
- **count**: The number of resources to create, used to control whether to create IAM role resources. Resources are only created when `var.role_id` is empty and `var.role_policy` is not empty
- **name**: The name of the IAM role, assigned by referencing the input variable `role_name`
- **type**: The type of the IAM role, assigned by referencing the input variable `role_type`, default is "XA" indicating a custom role
- **description**: The description of the IAM role, assigned by referencing the input variable `role_description`
- **policy**: The policy of the IAM role, assigned by referencing the input variable `role_policy`, the policy is a JSON format string

### 4. Create IAM User Group Resource

Add the following script to the TF file to instruct Terraform to create IAM user group resources:

```hcl
variable "group_id" {
  description = "The ID of the IAM group"
  type        = string
  default     = ""
  nullable    = false
}

variable "group_name" {
  description = "The name of the IAM group"
  type        = string
  default     = ""
  nullable    = false

  validation {
    condition     = !(var.group_name == "" && var.group_id == "")
    error_message = "The group_name must be provided when group_id is not provided."
  }
}

variable "group_description" {
  description = "The description of the IAM group"
  type        = string
  default     = ""
}

# Create IAM user group resource in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block)
resource "huaweicloud_identity_group" "test" {
  count = var.group_id == "" ? 1 : 0

  name        = var.group_name
  description = var.group_description
}
```

**Parameter Description**:
- **count**: The number of resources to create, used to control whether to create IAM user group resources. Resources are only created when `var.group_id` is empty
- **name**: The name of the IAM user group, assigned by referencing the input variable `group_name`
- **description**: The description of the IAM user group, assigned by referencing the input variable `group_description`

### 5. Query IAM Project Information Through Data Source

Add the following script to the TF file to instruct Terraform to perform a data source query, the query results are used to create IAM role assignment resources:

```hcl
variable "authorized_project_id" {
  description = "The ID of the IAM project"
  type        = string
  default     = ""
  nullable    = false
}

variable "authorized_project_name" {
  description = "The name of the IAM project"
  type        = string
  default     = ""
  nullable    = true

  validation {
    condition     = !(var.authorized_project_name == "" && var.authorized_project_id == "")
    error_message = "The authorized_project_name must be provided when authorized_project_id is not provided."
  }
}

# Query IAM project information that meets the conditions in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block), used to create IAM role assignment resources
data "huaweicloud_identity_projects" "test" {
  count = var.authorized_project_id == "" ? 1 : 0

  name = var.authorized_project_name
}
```

**Parameter Description**:
- **count**: The number of data sources to create, used to control whether to execute the IAM project query data source. The data source is only created (i.e., IAM project query is executed) when `var.authorized_project_id` is empty
- **name**: The name of the IAM project, assigned by referencing the input variable `authorized_project_name`

### 6. Create IAM Role Assignment Resource

Add the following script to the TF file to instruct Terraform to create IAM role assignment resources:

```hcl
variable "authorized_domain_id" {
  description = "The ID of the IAM domain"
  type        = string
  default     = ""
  nullable    = false
}

# Create IAM role assignment resource in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block)
resource "huaweicloud_identity_role_assignment" "test" {
  group_id   = var.group_id != "" ? var.group_id : huaweicloud_identity_group.test[0].id
  role_id    = var.role_id != "" ? var.role_id : var.role_policy != "" ? huaweicloud_identity_role.test[0].id : try(data.huaweicloud_identity_role.test[0].id, "NOT_FOUND")
  domain_id  = var.authorized_domain_id != "" ? var.authorized_domain_id : null
  project_id = var.authorized_domain_id == "" ? var.authorized_project_id != "" ? var.authorized_project_id : try(data.huaweicloud_identity_projects.test[0].projects[0].id, "NOT_FOUND") : null
}
```

**Parameter Description**:
- **group_id**: The ID of the IAM user group. If `var.group_id` is specified, use that value; otherwise, reference the ID of the IAM user group resource created earlier
- **role_id**: The ID of the IAM role. Based on variable configuration, choose to use `var.role_id`, the created IAM role resource ID, or the queried IAM role data source ID
- **domain_id**: The ID of the IAM domain. If `var.authorized_domain_id` is specified, use that value; otherwise, it is null, indicating authorization at the project level
- **project_id**: The ID of the IAM project. When `var.authorized_domain_id` is empty, if `var.authorized_project_id` is specified, use that value; otherwise, use the queried IAM project data source ID. When `var.authorized_domain_id` is not empty, this parameter is null, indicating authorization at the domain level

> Note: IAM role assignment supports authorization at the domain level or project level. When `domain_id` is specified, it indicates authorization at the domain level; when `project_id` is specified, it indicates authorization at the project level. The two cannot be specified simultaneously.

### 7. Create Random Password Resource

Add the following script to the TF file to instruct Terraform to create random password resources (automatically generated when users do not specify a password):

```hcl
# Create random password resource in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block), used to generate random passwords for users who have not specified a password
resource "random_password" "test" {
  count = length([for v in var.users_configuration : true if v.password == "" || v.password == null]) > 0 ? 1 : 0

  length           = 16
  special          = true
  override_special = "_%@"
}
```

**Parameter Description**:
- **count**: The number of resources to create, used to control whether to create random password resources. Resources are only created when there are users who have not specified a password
- **length**: The length of the password, set to 16 characters
- **special**: Whether to include special characters, set to true to include special characters
- **override_special**: The special character set, set to "_%@" indicating that the password can include underscore, percent sign, and @ symbol

### 8. Create IAM User Resource

Add the following script to the TF file to instruct Terraform to create IAM user resources:

```hcl
variable "users_configuration" {
  description = "The configuration of the IAM users"
  type        = list(object({
    name     = string
    password = optional(string, "")
  }))
  nullable    = false
}

# Create IAM user resource in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block)
resource "huaweicloud_identity_user" "test" {
  count = length(var.users_configuration)

  name     = lookup(var.users_configuration[count.index], "name", null)
  password = lookup(var.users_configuration[count.index], "password", null) != "" ? lookup(var.users_configuration[count.index], "password", null) : random_password.test[count.index].result
}
```

**Parameter Description**:
- **count**: The number of resources to create, creating the corresponding number of IAM user resources based on the length of the `var.users_configuration` list
- **name**: The name of the IAM user, obtained from the name field of the corresponding user in the `var.users_configuration` list
- **password**: The password of the IAM user. If a password is specified in the user configuration, use that password; otherwise, use the password generated by the random password resource

### 9. Create IAM User Group Membership Resource

Add the following script to the TF file to instruct Terraform to create IAM user group membership resources:

```hcl
# Create IAM user group membership resource in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block), adding users to the user group
resource "huaweicloud_identity_group_membership" "test" {
  group = var.group_id != "" ? var.group_id : huaweicloud_identity_group.test[0].id
  users = huaweicloud_identity_user.test[*].id
}
```

**Parameter Description**:
- **group**: The ID of the IAM user group. If `var.group_id` is specified, use that value; otherwise, reference the ID of the IAM user group resource created earlier
- **users**: The list of IAM user IDs, referencing the IDs of all IAM user resources created earlier, using the `[*]` syntax to get the IDs of all user resources

### 10. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# IAM role configuration
role_name        = "tf_test_role"
role_policy      = <<EOT
{
  "Version": "1.1",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "obs:*:*"
      ]
    },
    {
      "Effect": "Deny",
      "Action": [
        "obs:object:DeleteObjectVersion",
        "obs:object:DeleteAccessLabel",
        "obs:bucket:DeleteDirectColdAccessConfiguration",
        "obs:object:AbortMultipartUpload",
        "obs:bucket:DeleteBucketWebsite",
        "obs:object:DeleteObject",
        "obs:bucket:DeleteBucketPolicy",
        "obs:bucket:DeleteBucketCustomDomainConfiguration",
        "obs:object:RestoreObject",
        "obs:bucket:DeleteBucket",
        "obs:object:ModifyObjectMetaData",
        "obs:bucket:DeleteBucketInventoryConfiguration",
        "obs:bucket:DeleteReplicationConfiguration",
        "obs:bucket:DeleteBucketTagging"
      ]
    }
  ]
}
EOT
role_description = "Created by Terraform"

# IAM user group configuration
group_name = "tf_test_group"

# IAM project configuration
authorized_project_name = "cn-north-4"

# IAM user configuration
users_configuration = [
  {
    name = "tf_test_user"
  }
]
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command line parameters: `terraform apply -var="role_name=my-role" -var="group_name=my-group"`
2. Environment variables: `export TF_VAR_role_name=my-role`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 11. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating IAM roles, user groups, and users, and authorize users through user groups
4. Run `terraform show` to view the created IAM resource details

## Reference Information

- [Huawei Cloud IAM Product Documentation](https://support.huaweicloud.com/intl/en-us/iam/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For IAM Users Authorized Through Group](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/iam/users-authorized-through-group)
