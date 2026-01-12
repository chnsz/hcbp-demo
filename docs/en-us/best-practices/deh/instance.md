# Deploy Instance

## Application Scenario

Dedicated Host (DEH) is a physical server resource provided by Huawei Cloud, used to meet business scenarios with special requirements for resource exclusivity, security compliance, etc. By creating dedicated host instances, you can obtain full control of physical servers, achieve physical isolation of resources, and meet compliance requirements. Automating dedicated host instance creation through Terraform can ensure standardized and consistent resource deployment, improving operational efficiency. This best practice will introduce how to use Terraform to automatically create dedicated host instances.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Data Sources

- [Availability Zones Data Source (huaweicloud_availability_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [Dedicated Host Types Data Source (huaweicloud_deh_types)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/deh_types)

### Resources

- [Dedicated Host Instance Resource (huaweicloud_deh_instance)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/deh_instance)

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Query Data Sources

Add the following script to the TF file (e.g., main.tf) to query availability zone and dedicated host type information:

```hcl
# Query availability zone information
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}

# Query dedicated host type information
data "huaweicloud_deh_types" "test" {
  count = var.deh_instance_host_type == "" ? 1 : 0

  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
}
```

**Parameter Description**:
- **availability_zone**: Availability zone name, queried when availability_zone variable is empty
- **host_type**: Dedicated host type, queried when deh_instance_host_type variable is empty

### 3. Create Dedicated Host Instance Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a dedicated host instance resource:

```hcl
variable "instance_name" {
  description = "The name of the dedicated host instance"
  type        = string
}

variable "availability_zone" {
  description = "The availability zone where the resources will be created"
  type        = string
  default     = ""
}

variable "deh_instance_host_type" {
  description = "The host type of the dedicated host"
  type        = string
  default     = ""
}

variable "deh_instance_auto_placement" {
  description = "Whether to enable auto placement for the dedicated host"
  type        = string
  default     = "on"
}

variable "enterprise_project_id" {
  description = "The enterprise project ID of the dedicated host"
  type        = string
  default     = null
}

variable "deh_instance_charging_mode" {
  description = "The charging mode of the dedicated host"
  type        = string
  default     = "prePaid"
}

variable "deh_instance_period_unit" {
  description = "The unit of the billing period of the dedicated host"
  type        = string
  default     = "month"
}

variable "deh_instance_period" {
  description = "The billing period of the dedicated host"
  type        = string
  default     = "1"
}

variable "deh_instance_auto_renew" {
  description = "Whether to enable auto renew for the dedicated host"
  type        = string
  default     = "false"
}

# Create dedicated host instance resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_deh_instance" "test" {
  name                  = var.instance_name
  availability_zone     = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  host_type             = var.deh_instance_host_type != "" ? var.deh_instance_host_type : try(data.huaweicloud_deh_types.test[0].dedicated_host_types[0].host_type, null)
  auto_placement        = var.deh_instance_auto_placement
  enterprise_project_id = var.enterprise_project_id
  charging_mode         = var.deh_instance_charging_mode
  period_unit           = var.deh_instance_period_unit
  period                = var.deh_instance_period
  auto_renew            = var.deh_instance_auto_renew
}
```

**Parameter Description**:
- **name**: Dedicated host instance name, assigned by referencing the input variable instance_name
- **availability_zone**: Availability zone name, assigned by referencing the input variable availability_zone or availability zones data source
- **host_type**: Dedicated host type, assigned by referencing the input variable deh_instance_host_type or dedicated host types data source
- **auto_placement**: Whether to enable auto placement, assigned by referencing the input variable deh_instance_auto_placement, default value is "on"
- **enterprise_project_id**: Enterprise project ID, assigned by referencing the input variable enterprise_project_id, optional parameter, default value is null
- **charging_mode**: Charging mode, assigned by referencing the input variable deh_instance_charging_mode, default value is "prePaid" (prepaid)
- **period_unit**: Billing period unit, assigned by referencing the input variable deh_instance_period_unit, default value is "month"
- **period**: Billing period, assigned by referencing the input variable deh_instance_period, default value is "1"
- **auto_renew**: Whether to enable auto renew, assigned by referencing the input variable deh_instance_auto_renew, default value is "false"

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Dedicated Host Instance Configuration
instance_name = "tf_test_deh_instance"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="instance_name=my_deh_instance"`
2. Environment variables: `export TF_VAR_instance_name=my_deh_instance`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create the dedicated host instance:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the dedicated host instance
4. Run `terraform show` to view the details of the created dedicated host instance

> Note: After the dedicated host instance is created, ECS instances can be deployed on this dedicated host. Dedicated Host provides full control of physical servers, achieves physical isolation of resources, and meets compliance requirements. By setting auto_placement, you can enable the auto placement function, and the system will automatically select appropriate physical servers.

## Reference Information

- [Huawei Cloud DEH Product Documentation](https://support.huaweicloud.com/deh/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Instance](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/deh/instance)
