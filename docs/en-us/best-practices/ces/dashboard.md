# Deploy Dashboard

## Application Scenario

Cloud Eye Service (CES) dashboard is a monitoring data visualization function provided by the CES service, used to centrally display multiple monitoring metrics and resource status. By configuring dashboards, you can create custom monitoring views, centrally display multiple monitoring metrics in the form of charts, tables, etc., making it convenient for operation personnel to quickly understand the overall running status of cloud resources. Automating CES dashboard creation through Terraform can ensure standardized and consistent monitoring view configuration, improving operational efficiency. This best practice will introduce how to use Terraform to automatically create CES dashboards.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [CES Dashboard Resource (huaweicloud_ces_dashboard)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ces_dashboard)

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create CES Dashboard Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a CES dashboard resource:

```hcl
variable "dashboard_name" {
  description = "The name of the alarm dashboard"
  type        = string
}

variable "dashboard_row_widget_num" {
  description = "The monitoring view display mode"
  type        = number
}

variable "dashboard_extend_info" {
  description = "The information about the extension"
  type = list(object({
    filter                  = string
    period                  = string
    display_time            = number
    refresh_time            = number
    from                    = number
    to                      = number
    screen_color            = string
    enable_screen_auto_play = bool
    time_interval           = number
    enable_legend           = bool
    full_screen_widget_num  = number
  }))
  default = []
}

variable "dashboard_id" {
  description = "The copied dashboard ID"
  type        = string
  default     = null
}

variable "enterprise_project_id" {
  description = "The enterprise project ID of the dashboard"
  type        = string
  default     = null
}

variable "is_favorite" {
  description = "Whether the dashboard is favorite"
  type        = bool
  default     = false
}

# Create CES dashboard resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_ces_dashboard" "test" {
  name           = var.dashboard_name
  row_widget_num = var.dashboard_row_widget_num

  dynamic "extend_info" {
    for_each = length(var.dashboard_extend_info) > 0 ? var.dashboard_extend_info : []

    content {
      filter                  = extend_info.value.filter
      period                  = extend_info.value.period
      display_time            = extend_info.value.display_time
      refresh_time            = extend_info.value.refresh_time
      from                    = extend_info.value.from
      to                      = extend_info.value.to
      screen_color            = extend_info.value.screen_color
      enable_screen_auto_play = extend_info.value.enable_screen_auto_play
      time_interval           = extend_info.value.time_interval
      enable_legend           = extend_info.value.enable_legend
      full_screen_widget_num  = extend_info.value.full_screen_widget_num
    }
  }

  dashboard_id         = var.dashboard_id
  enterprise_project_id = var.enterprise_project_id
  is_favorite         = var.is_favorite
}
```

**Parameter Description**:
- **name**: The dashboard name, assigned by referencing the input variable dashboard_name
- **row_widget_num**: The monitoring view display mode, assigned by referencing the input variable dashboard_row_widget_num
- **extend_info**: The extension information list, assigned by referencing the input variable dashboard_extend_info, optional parameter, default value is empty list, each extension information contains the following parameters:
  - **filter**: Metric aggregation method
  - **period**: Metric aggregation period
  - **display_time**: Display time
  - **refresh_time**: Refresh time
  - **from**: Start time
  - **to**: End time
  - **screen_color**: Monitoring screen background color
  - **enable_screen_auto_play**: Whether to enable monitoring screen automatic switching
  - **time_interval**: Monitoring screen automatic switching time interval
  - **enable_legend**: Whether to enable legend
  - **full_screen_widget_num**: Number of large screen display views
- **dashboard_id**: The copied dashboard ID, assigned by referencing the input variable dashboard_id, optional parameter, default value is null
- **enterprise_project_id**: The enterprise project ID of the dashboard, assigned by referencing the input variable enterprise_project_id, optional parameter, default value is null
- **is_favorite**: Whether the dashboard is favorite, assigned by referencing the input variable is_favorite, optional parameter, default value is false

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Dashboard Configuration
dashboard_name        = "tf_test_ces_dashboard"
dashboard_row_widget_num = 2

# Extension Information Configuration
dashboard_extend_info = [
  {
    filter                  = "average"
    period                  = "1"
    display_time            = 3600
    refresh_time            = 60
    from                    = 0
    to                      = 3600
    screen_color            = "#000000"
    enable_screen_auto_play = true
    time_interval           = 30
    enable_legend           = true
    full_screen_widget_num   = 4
  }
]

# Optional Configuration
# dashboard_id         = "existing_dashboard_id"
# enterprise_project_id = "enterprise_project_id"
# is_favorite          = true
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="dashboard_name=my_dashboard" -var="dashboard_row_widget_num=2"`
2. Environment variables: `export TF_VAR_dashboard_name=my_dashboard` and `export TF_VAR_dashboard_row_widget_num=2`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create a CES dashboard:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the dashboard
4. Run `terraform show` to view the details of the created dashboard

> Note: After the dashboard is created, you can further configure it in the CES console, adding monitoring metrics and charts. By setting extend_info, you can configure the display parameters of the monitoring screen, including automatic switching, refresh time, etc. The dashboard supports favorite function, making it convenient to quickly access commonly used monitoring views.

## Reference Information

- [Huawei Cloud CES Product Documentation](https://support.huaweicloud.com/ces/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Dashboard](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ces/dashboard)
