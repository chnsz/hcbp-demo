# Deploy Migration Task

## Application Scenario

Server Migration Service (SMS) is a server migration service provided by Huawei Cloud, supporting migration of physical servers, virtual machines, or servers from other cloud platforms to Huawei Cloud, achieving seamless business migration. Migration tasks are core resources of SMS service, used to execute specific server migration operations. By creating migration tasks, parameters such as migration type, operating system type, source server, and target template can be configured, achieving automated server migration. This best practice introduces how to use Terraform to automatically deploy migration tasks, including querying availability zones and source servers, creating server templates, and creating migration tasks.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [Availability Zones Query Data Source (data.huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [Source Servers Query Data Source (data.huaweicloud_sms_source_servers)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/sms_source_servers)

### Resources

- [Server Template Resource (huaweicloud_sms_server_template)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sms_server_template)
- [Migration Task Resource (huaweicloud_sms_task)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/sms_task)

### Resource/Data Source Dependencies

```text
data.huaweicloud_availability_zones
    └── huaweicloud_sms_server_template

data.huaweicloud_sms_source_servers
    └── huaweicloud_sms_task

huaweicloud_sms_server_template
    └── huaweicloud_sms_task
```

> Note: Migration tasks depend on source servers and server templates. Server templates need to specify availability zones for creating target servers. Migration tasks configure migration parameters by referencing source server IDs and server template IDs.

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration details, refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../../introductions/prepare_before_deploy.md).

### 2. Query Availability Zones List

Add the following script to the TF file (such as main.tf) to query the availability zones list:

```hcl
# Query availability zones list data source
data "huaweicloud_availability_zones" "test" {}
```

**Parameter Description**:
- This data source does not require input parameters and will automatically query all availability zones in the current region

> Note: The availability zones list is used to specify availability zones when creating server templates later.

### 3. Query Source Servers List

Add the following script to the TF file (such as main.tf) to query the source servers list:

```hcl
variable "source_server_name" {
  description = "The name of the SMS source server"
  type        = string
  default     = null
}

# Query source servers list data source
data "huaweicloud_sms_source_servers" "test" {
  name = var.source_server_name
}
```

**Parameter Description**:
- **name**: Source server name, assigned by referencing the input variable `source_server_name`, optional parameter, used to filter source servers

> Note: The source servers list is used to obtain information about source servers that need to be migrated. The source server ID needs to be referenced when creating migration tasks later.

### 4. Create Server Template

Add the following script to the TF file (such as main.tf) to create a server template:

```hcl
variable "server_template_name" {
  description = "The name of the SMS server template"
  type        = string
}

# Create server template resource in the specified region (defaults to the region specified in the current provider block when region parameter is omitted)
resource "huaweicloud_sms_server_template" "test" {
  name              = var.server_template_name
  availability_zone = try(data.huaweicloud_availability_zones.test.names[0], null)
}
```

**Parameter Description**:
- **name**: Server template name, assigned by referencing the input variable `server_template_name`
- **availability_zone**: Availability zone, assigned by referencing the first availability zone name from the availability zones list data source, using `try` function to handle possible null values

> Note: Server templates are used to define target server configurations, including availability zones, specifications, and other information. Server template IDs need to be referenced when creating migration tasks.

### 5. Create Migration Task

Add the following script to the TF file (such as main.tf) to create a migration task:

```hcl
variable "migrate_task_type" {
  description = "The type of the SMS task"
  type        = string
}

variable "server_os_type" {
  description = "The OS type of the server"
  type        = string
}

# Create migration task resource in the specified region (defaults to the region specified in the current provider block when region parameter is omitted)
resource "huaweicloud_sms_task" "test" {
  type             = var.migrate_task_type
  os_type          = var.server_os_type
  source_server_id = try(data.huaweicloud_sms_source_servers.test.servers[0].id, null)
  vm_template_id   = huaweicloud_sms_server_template.test.id
}
```

**Parameter Description**:
- **type**: Migration task type, assigned by referencing the input variable `migrate_task_type`, supports `MIGRATE_BLOCK` (block-level migration) and `MIGRATE_FILE` (file-level migration)
- **os_type**: Server operating system type, assigned by referencing the input variable `server_os_type`, supports `WINDOWS` and `LINUX`
- **source_server_id**: Source server ID, assigned by referencing the first server ID from the source servers list data source, using `try` function to handle possible null values
- **vm_template_id**: Virtual machine template ID, assigned by referencing the server template resource ID

> Note: Migration tasks are used to execute specific server migration operations. Migration task types need to be selected according to actual requirements. Block-level migration is suitable for whole machine migration, and file-level migration is suitable for file migration. Source server ID and virtual machine template ID are required parameters.

### 6. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input each time.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Source server configuration (Optional)
source_server_name = "tf_source_server_name"

# Server template configuration (Required)
server_template_name = "tf_server_template_name"

# Migration task configuration (Required)
migrate_task_type = "MIGRATE_BLOCK"
server_os_type    = "WINDOWS"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows Terraform to automatically import the variable values in this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command-line parameters: `terraform apply -var="server_template_name=tf_server_template_name" -var="migrate_task_type=MIGRATE_BLOCK"`
2. Environment variables: `export TF_VAR_server_template_name=tf_server_template_name`
3. Custom-named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command-line parameters > variable files > environment variables > default values.

### 7. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating server template and migration task
4. Run `terraform show` to view the created migration task

## Reference Information

- [Huawei Cloud SMS Product Documentation](https://support.huaweicloud.com/sms/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Migration Task](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/sms/migration-task)
