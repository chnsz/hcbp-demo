# Deploy Alarm Template

## Application Scenario

Cloud Eye Service (CES) alarm template is an alarm rule template function provided by the CES service, used to quickly create and manage alarm rules. By configuring alarm templates, you can define unified alarm policies, including monitoring metrics, alarm thresholds, alarm levels, etc., and then quickly create multiple alarm rules based on templates, improving the efficiency and consistency of alarm configuration. Automating CES alarm template creation through Terraform can ensure standardized and standardized alarm configuration, simplifying operational management. This best practice will introduce how to use Terraform to automatically create CES alarm templates.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [CES Alarm Template Resource (huaweicloud_ces_alarm_template)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ces_alarm_template)

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create CES Alarm Template Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a CES alarm template resource:

```hcl
variable "alarm_template_name" {
  description = "The name of the alarm template"
  type        = string
}

variable "alarm_template_description" {
  description = "The description of the alarm template"
  type        = string
  default     = ""
}

variable "alarm_template_policies" {
  description = "The policy list of the CES alarm template"
  type = list(object({
    namespace           = string
    metric_name         = string
    period              = number
    filter              = string
    comparison_operator = string
    count               = number
    suppress_duration   = number
    value               = number
    alarm_level         = number
    unit                = string
    dimension_name      = string
    hierarchical_value = list(object({
      critical = number
      major    = number
      minor    = number
      info     = number
    }))
  }))
  default = []
}

# Create CES alarm template resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_ces_alarm_template" "test" {
  name        = var.alarm_template_name
  description = var.alarm_template_description

  dynamic "policies" {
    for_each = var.alarm_template_policies

    content {
      namespace           = policies.value.namespace
      metric_name         = policies.value.metric_name
      period              = policies.value.period
      filter              = policies.value.filter
      comparison_operator = policies.value.comparison_operator
      count               = policies.value.count
      suppress_duration   = policies.value.suppress_duration
      value               = policies.value.value
      alarm_level         = policies.value.alarm_level
      unit                = policies.value.unit
      dimension_name      = policies.value.dimension_name
      hierarchical_value {
        critical = policies.value.hierarchical_value[0].critical
        major    = policies.value.hierarchical_value[0].major
        minor    = policies.value.hierarchical_value[0].minor
        info     = policies.value.hierarchical_value[0].info
      }
    }
  }
}
```

**Parameter Description**:
- **name**: The alarm template name, assigned by referencing the input variable alarm_template_name
- **description**: The alarm template description, assigned by referencing the input variable alarm_template_description, optional parameter, default value is empty string
- **policies**: The alarm policy list, assigned by referencing the input variable alarm_template_policies, each policy contains the following parameters:
  - **namespace**: Service namespace, such as SYS.APIG, SYS.ECS, etc.
  - **metric_name**: Alarm metric name
  - **period**: Alarm condition judgment period, unit is minutes
  - **filter**: Data aggregation method, such as average (average value), max (maximum value), min (minimum value), etc.
  - **comparison_operator**: Comparison condition for alarm threshold, such as >, >=, <, <=, =, etc.
  - **count**: Number of consecutive alarm triggering times
  - **suppress_duration**: Alarm suppression cycle, unit is seconds
  - **value**: Alarm threshold
  - **alarm_level**: Alarm level, 1-4 represent critical, major, minor, info respectively
  - **unit**: Unit string of alarm threshold
  - **dimension_name**: Resource dimension name
  - **hierarchical_value**: Multi-level alarm threshold configuration, contains the following fields:
    - **critical**: Threshold for critical level
    - **major**: Threshold for major level
    - **minor**: Threshold for minor level
    - **info**: Threshold for info level

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Alarm Template Configuration
alarm_template_name        = "tf_test_alarm_template"
alarm_template_description = "Test alarm template for APIG service"

# Alarm Policy Configuration
alarm_template_policies = [
  {
    namespace           = "SYS.APIG"
    dimension_name      = "api_id"
    metric_name         = "req_count_2xx"
    period              = 1
    filter              = "average"
    comparison_operator = ">"
    value               = 10
    unit                = "times/minute"
    count               = 3
    alarm_level         = 2
    suppress_duration   = 300
    hierarchical_value = [
      {
        critical = 12
        major    = 10
        minor    = 8
        info     = 4
      }
    ]
  }
]
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="alarm_template_name=my_template" -var="alarm_template_description=My description"`
2. Environment variables: `export TF_VAR_alarm_template_name=my_template` and `export TF_VAR_alarm_template_description=My description`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create a CES alarm template:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the alarm template
4. Run `terraform show` to view the details of the created alarm template

> Note: After the alarm template is created, multiple alarm rules can be quickly created based on this template. The hierarchical_value in the alarm policy is used to configure multi-level alarm thresholds, and different thresholds can be set according to different alarm levels. Alarm levels 1-4 represent critical, major, minor, and info respectively, and you can choose the appropriate alarm level according to business needs.

## Reference Information

- [Huawei Cloud CES Product Documentation](https://support.huaweicloud.com/ces/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Alarm Template](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ces/alarm-template)
