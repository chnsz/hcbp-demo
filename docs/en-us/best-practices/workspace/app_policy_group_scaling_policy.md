# Deploy Cloud Application Policy Group Scaling Policy

## Application Scenario

Huawei Cloud Cloud Desktop (Workspace) is a cloud computing-based desktop virtualization service that provides enterprise users with secure and convenient cloud office solutions. Cloud application policy group scaling policies are an important component of the Workspace service's cloud application functionality, used to configure automatic scaling policies for cloud application server groups, automatically adjusting the number of server instances based on session usage to achieve elastic resource expansion and cost optimization.

Through cloud application policy group scaling policies, enterprises can automatically adjust the number of instances in cloud application server groups based on actual business load. When session usage exceeds the threshold, the system automatically scales out, and when session idle time reaches the set value, the system automatically scales in. This automatic scaling mechanism helps enterprises achieve on-demand resource allocation, improve resource utilization, reduce operating costs, and ensure user access experience. This best practice will introduce how to use Terraform to automatically deploy Workspace cloud application policy group scaling policies, including cloud application server group creation, cloud application group creation, policy group configuration, and scaling policy configuration.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Workspace Service Query Data Source (data.huaweicloud_workspace_service)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/workspace_service)

### Resources

- [Workspace Application Server Group Resource (huaweicloud_workspace_app_server_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_app_server_group)
- [Workspace Application Group Resource (huaweicloud_workspace_app_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_app_group)
- [Workspace Application Policy Group Resource (huaweicloud_workspace_app_policy_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_app_policy_group)
- [Workspace Application Server Group Scaling Policy Resource (huaweicloud_workspace_app_server_group_scaling_policy)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_app_server_group_scaling_policy)

### Resource/Data Source Dependencies

```
data.huaweicloud_workspace_service.test
    └── huaweicloud_workspace_app_server_group.test
        ├── huaweicloud_workspace_app_group.test
        │   └── huaweicloud_workspace_app_policy_group.test
        └── huaweicloud_workspace_app_server_group_scaling_policy.test
```

## Implementation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) for configuration introduction.

### 2. Query Workspace Service Information Through Data Source

Add the following script to the TF file (such as main.tf) to instruct Terraform to perform a data source query, the query results are used to create cloud application server groups:

```hcl
# Query all Workspace service information in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block), used to create cloud application server groups
data "huaweicloud_workspace_service" "test" {}
```

**Parameter Description**:
- This data source requires no additional parameters and automatically queries Workspace service information in the current region

### 3. Create Workspace Cloud Application Server Group

Add the following script to the TF file (such as main.tf) to instruct Terraform to create cloud application server group resources:

```hcl
variable "app_server_group_name" {
  description = "The name of the APP server group"
  type        = string
}

variable "app_server_group_app_type" {
  description = "The application type of the APP server group"
  type        = string
  default     = "SESSION_DESKTOP_APP"
}

variable "app_server_group_os_type" {
  description = "The operating system type of the APP server group"
  type        = string
  default     = "Windows"
}

variable "app_server_group_flavor_id" {
  description = "The flavor ID of the APP server group"
  type        = string
}

variable "app_server_group_image_id" {
  description = "The image ID of the APP server group"
  type        = string
}

variable "app_server_group_image_product_id" {
  description = "The image product ID of the APP server group"
  type        = string
}

variable "app_server_group_system_disk_type" {
  description = "The system disk type of the APP server group"
  type        = string
  default     = "SAS"
}

variable "app_server_group_system_disk_size" {
  description = "The system disk size of the APP server group in GB"
  type        = number
  default     = 80

  validation {
    condition     = var.app_server_group_system_disk_size >= 40 && var.app_server_group_system_disk_size <= 2048
    error_message = "The system disk size must be between 40 and 2048 GB."
  }
}

# Create Workspace cloud application server group resource in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block)
resource "huaweicloud_workspace_app_server_group" "test" {
  name             = var.app_server_group_name
  app_type         = var.app_server_group_app_type
  os_type          = var.app_server_group_os_type
  flavor_id        = var.app_server_group_flavor_id
  image_type       = "gold"
  image_id         = var.app_server_group_image_id
  image_product_id = var.app_server_group_image_product_id
  vpc_id           = data.huaweicloud_workspace_service.test.vpc_id
  subnet_id        = try(data.huaweicloud_workspace_service.test.network_ids[0], null)
  system_disk_type = var.app_server_group_system_disk_type
  system_disk_size = var.app_server_group_system_disk_size
  is_vdi           = true
}
```

**Parameter Description**:
- **name**: Cloud application server group name, assigned by referencing the input variable `app_server_group_name`
- **app_type**: Application type, assigned by referencing the input variable `app_server_group_app_type`, default is "SESSION_DESKTOP_APP"
- **os_type**: Operating system type, assigned by referencing the input variable `app_server_group_os_type`, default is "Windows"
- **flavor_id**: Flavor ID, assigned by referencing the input variable `app_server_group_flavor_id`
- **image_type**: Image type, fixed as "gold" (golden image)
- **image_id**: Image ID, assigned by referencing the input variable `app_server_group_image_id`
- **image_product_id**: Image product ID, assigned by referencing the input variable `app_server_group_image_product_id`
- **vpc_id**: VPC ID, assigned based on the return results of the Workspace service query data source (data.huaweicloud_workspace_service)
- **subnet_id**: Subnet ID, assigned based on the return results of the Workspace service query data source (data.huaweicloud_workspace_service)
- **system_disk_type**: System disk type, assigned by referencing the input variable `app_server_group_system_disk_type`, default is "SAS"
- **system_disk_size**: System disk size, assigned by referencing the input variable `app_server_group_system_disk_size`, default is 80GB
- **is_vdi**: Whether it is VDI mode, fixed as true

### 4. Create Workspace Cloud Application Group

Add the following script to the TF file to instruct Terraform to create cloud application group resources:

```hcl
variable "app_group_name" {
  description = "The name of the APP group"
  type        = string
}

# Create Workspace cloud application group resource in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block)
resource "huaweicloud_workspace_app_group" "test" {
  depends_on = [huaweicloud_workspace_app_server_group.test]

  server_group_id = huaweicloud_workspace_app_server_group.test.id
  name            = var.app_group_name
  type            = "SESSION_DESKTOP_APP"
  description     = "Created APP group by Terraform"
}
```

**Parameter Description**:
- **depends_on**: Explicit dependency declaration, ensuring that the cloud application server group is created before creating the cloud application group
- **server_group_id**: The ID of the cloud application server group, referencing the ID of the cloud application server group resource created earlier
- **name**: Cloud application group name, assigned by referencing the input variable `app_group_name`
- **type**: Cloud application group type, fixed as "SESSION_DESKTOP_APP" indicating a session desktop application group
- **description**: Cloud application group description, fixed as "Created APP group by Terraform"

### 5. Create Workspace Cloud Application Policy Group

Add the following script to the TF file to instruct Terraform to create cloud application policy group resources:

```hcl
variable "policy_group_name" {
  description = "The name of the policy group"
  type        = string
}

variable "policy_group_priority" {
  description = "The priority of the policy group"
  type        = number
  default     = 1
}

variable "policy_group_description" {
  description = "The description of the policy group"
  type        = string
  default     = "Created APP policy group by Terraform"
}

variable "target_type" {
  description = "The type of target for the policy group"
  type        = string
  default     = "APPGROUP"

  validation {
    condition     = contains(["APPGROUP", "ALL"], var.target_type)
    error_message = "The target_type must be either 'APPGROUP' or 'ALL'."
  }
}

variable "automatic_reconnection_interval" {
  description = "The automatic reconnection interval in minutes"
  type        = number
  default     = 10

  validation {
    condition     = var.automatic_reconnection_interval >= 1 && var.automatic_reconnection_interval <= 60
    error_message = "The automatic_reconnection_interval must be between 1 and 60 minutes."
  }
}

variable "session_persistence_time" {
  description = "The session persistence time in minutes"
  type        = number
  default     = 120

  validation {
    condition     = var.session_persistence_time >= 1 && var.session_persistence_time <= 1440
    error_message = "The session_persistence_time must be between 1 and 1440 minutes."
  }
}

variable "forbid_screen_capture" {
  description = "Whether to forbid screen capture"
  type        = bool
  default     = true
}

# Create Workspace cloud application policy group resource in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block)
resource "huaweicloud_workspace_app_policy_group" "test" {
  depends_on = [huaweicloud_workspace_app_group.test]

  name        = var.policy_group_name
  priority    = var.policy_group_priority
  description = var.policy_group_description

  targets {
    id   = var.target_type == "APPGROUP" ? huaweicloud_workspace_app_group.test.id : "default-apply-all-targets"
    name = var.target_type == "APPGROUP" ? huaweicloud_workspace_app_group.test.name : "All-Targets"
    type = var.target_type
  }

  policies = jsonencode({
    "client": {
      "automatic_reconnection_interval": var.automatic_reconnection_interval,
      "session_persistence_time":        var.session_persistence_time,
      "forbid_screen_capture":           var.forbid_screen_capture
    }
  })
}
```

**Parameter Description**:
- **depends_on**: Explicit dependency declaration, ensuring that the cloud application group is created before creating the policy group
- **name**: Policy group name, assigned by referencing the input variable `policy_group_name`
- **priority**: Policy group priority, assigned by referencing the input variable `policy_group_priority`, default is 1, smaller values indicate higher priority
- **description**: Policy group description, assigned by referencing the input variable `policy_group_description`, default is "Created APP policy group by Terraform"
- **targets**: Policy group target configuration block
  - **id**: Target ID, if target type is "APPGROUP" then use the cloud application group ID, otherwise use "default-apply-all-targets" to indicate applying to all targets
  - **name**: Target name, if target type is "APPGROUP" then use the cloud application group name, otherwise use "All-Targets"
  - **type**: Target type, assigned by referencing the input variable `target_type`, default is "APPGROUP" indicating applying to specified application group, "ALL" indicating applying to all targets
- **policies**: Policy configuration, using jsonencode function to encode policy configuration as JSON string
  - **client.automatic_reconnection_interval**: Client automatic reconnection interval (minutes), assigned by referencing the input variable `automatic_reconnection_interval`, default is 10 minutes
  - **client.session_persistence_time**: Session persistence time (minutes), assigned by referencing the input variable `session_persistence_time`, default is 120 minutes
  - **client.forbid_screen_capture**: Whether to forbid screen capture, assigned by referencing the input variable `forbid_screen_capture`, default is true

### 6. Create Workspace Cloud Application Server Group Scaling Policy

Add the following script to the TF file to instruct Terraform to create cloud application server group scaling policy resources:

```hcl
variable "max_scaling_amount" {
  description = "The maximum number of instances that can be scaled out"
  type        = number

  validation {
    condition     = var.max_scaling_amount >= 1 && var.max_scaling_amount <= 100
    error_message = "The max_scaling_amount must be between 1 and 100."
  }
}

variable "single_expansion_count" {
  description = "The number of instances to scale out in a single scaling operation"
  type        = number

  validation {
    condition     = var.single_expansion_count >= 1 && var.single_expansion_count <= 10
    error_message = "The single_expansion_count must be between 1 and 10."
  }
}

variable "session_usage_threshold" {
  description = "The session usage threshold percentage"
  type        = number
  default     = 80

  validation {
    condition     = var.session_usage_threshold >= 1 && var.session_usage_threshold <= 100
    error_message = "The session_usage_threshold must be between 1 and 100."
  }
}

variable "shrink_after_session_idle_minutes" {
  description = "The number of minutes to wait before shrinking idle instances"
  type        = number
  default     = 30

  validation {
    condition     = var.shrink_after_session_idle_minutes >= 1 && var.shrink_after_session_idle_minutes <= 1440
    error_message = "The shrink_after_session_idle_minutes must be between 1 and 1440 minutes."
  }
}

# Create Workspace cloud application server group scaling policy resource in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block)
resource "huaweicloud_workspace_app_server_group_scaling_policy" "test" {
  depends_on = [huaweicloud_workspace_app_server_group.test]

  server_group_id        = huaweicloud_workspace_app_server_group.test.id
  max_scaling_amount     = var.max_scaling_amount
  single_expansion_count = var.single_expansion_count

  scaling_policy_by_session {
    session_usage_threshold           = var.session_usage_threshold
    shrink_after_session_idle_minutes = var.shrink_after_session_idle_minutes
  }
}
```

**Parameter Description**:
- **depends_on**: Explicit dependency declaration, ensuring that the cloud application server group is created before creating the scaling policy
- **server_group_id**: The ID of the cloud application server group, referencing the ID of the cloud application server group resource created earlier
- **max_scaling_amount**: The maximum number of instances that can be scaled out, assigned by referencing the input variable `max_scaling_amount`, valid range is 1 to 100
- **single_expansion_count**: The number of instances to scale out in a single scaling operation, assigned by referencing the input variable `single_expansion_count`, valid range is 1 to 10
- **scaling_policy_by_session**: Scaling policy configuration block based on sessions
  - **session_usage_threshold**: Session usage threshold (percentage), assigned by referencing the input variable `session_usage_threshold`, default is 80%. When session usage exceeds this threshold, scaling out is triggered
  - **shrink_after_session_idle_minutes**: The number of minutes to wait before shrinking idle instances, assigned by referencing the input variable `shrink_after_session_idle_minutes`, default is 30 minutes. When session idle time reaches this value, scaling in is triggered

> Note: The scaling policy automatically adjusts based on session usage. When session usage exceeds the threshold, the system automatically scales out instances; when session idle time reaches the set value, the system automatically scales in instances. This enables elastic resource expansion and cost optimization.

### 7. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Cloud application server group configuration
app_server_group_name             = "tf_test_server_group"
app_server_group_flavor_id        = "workspace.appstream.general.xlarge.4"
app_server_group_image_id         = "2ac7b1fb-b198-422b-a45f-61ea285cb6e7"
app_server_group_image_product_id = "OFFI886188719633408000"

# Cloud application group configuration
app_group_name = "tf_test_app_group"

# Policy group configuration
policy_group_name = "tf_test_policy_group"

# Scaling policy configuration
max_scaling_amount     = 10
single_expansion_count = 2
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command line parameters: `terraform apply -var="app_server_group_name=my-server-group" -var="max_scaling_amount=10"`
2. Environment variables: `export TF_VAR_app_server_group_name=my-server-group`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 8. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating cloud application server groups, cloud application groups, cloud application policy groups, and scaling policies
4. Run `terraform show` to view the created cloud application policy group scaling policy details

## Reference Information

- [Huawei Cloud Workspace Product Documentation](https://support.huaweicloud.com/intl/en-us/workspace/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Workspace Cloud Application Policy Group Scaling Policy](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/workspace/app/policy_group_scaling_policy)
