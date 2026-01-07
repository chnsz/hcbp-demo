# Deploy Anti-DDoS LTS Configuration

## Application Scenario

Anti-DDoS (Anti-Distributed Denial of Service) is a distributed denial-of-service attack protection service provided by Huawei Cloud, which can effectively protect public IPs from DDoS attacks and ensure stable business operations. By configuring the integration between Anti-DDoS and Log Tank Service (LTS), Anti-DDoS attack logs can be transmitted to LTS in real-time, facilitating log analysis, auditing, and monitoring. LTS configuration helps users centrally manage and analyze Anti-DDoS protection logs, enabling timely detection and response to security threats.

This best practice will introduce how to use Terraform to automatically deploy Anti-DDoS LTS configuration, including creating LTS log groups, log streams, and configuring the integration between Anti-DDoS and LTS.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Log Tank Service Log Group Resource (huaweicloud_lts_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [Log Tank Service Log Stream Resource (huaweicloud_lts_stream)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)
- [Anti-DDoS LTS Configuration Resource (huaweicloud_antiddos_lts_config)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/antiddos_lts_config)

### Resource/Data Source Dependencies

```
huaweicloud_lts_group
    └── huaweicloud_lts_stream
        └── huaweicloud_antiddos_lts_config
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create Log Tank Service Log Group Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a Log Tank Service log group resource:

```hcl
variable "lts_group_name" {
  description = "The name of the LTS group"
  type        = string
}

variable "lts_ttl_in_days" {
  description = "The log expiration time(days)"
  type        = number
}

variable "enterprise_project_id" {
  description = "The enterprise project ID"
  type        = string
  default     = null
}

# Create Log Tank Service log group resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_lts_group" "test" {
  group_name            = var.lts_group_name
  ttl_in_days           = var.lts_ttl_in_days
  enterprise_project_id = var.enterprise_project_id
}
```

**Parameter Description**:
- **group_name**: The log group name, assigned by referencing the input variable lts_group_name
- **ttl_in_days**: The log retention time (unit: days), assigned by referencing the input variable lts_ttl_in_days
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id, default value is null

### 3. Create Log Tank Service Log Stream Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a Log Tank Service log stream resource:

```hcl
variable "lts_stream_name" {
  description = "The name of the LTS stream"
  type        = string
}

variable "lts_is_favorite" {
  description = "Whether to favorite the log stream"
  type        = bool
  default     = false
}

# Create Log Tank Service log stream resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_lts_stream" "test" {
  group_id              = huaweicloud_lts_group.test.id
  stream_name           = var.lts_stream_name
  is_favorite           = var.lts_is_favorite
  enterprise_project_id = var.enterprise_project_id
}
```

**Parameter Description**:
- **group_id**: The log group ID, referencing the ID of the previously created Log Tank Service log group resource (huaweicloud_lts_group.test)
- **stream_name**: The log stream name, assigned by referencing the input variable lts_stream_name
- **is_favorite**: Whether to favorite the log stream, assigned by referencing the input variable lts_is_favorite, default value is false
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id, default value is null

### 4. Create Anti-DDoS LTS Configuration Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an Anti-DDoS LTS configuration resource:

```hcl
# Create Anti-DDoS LTS configuration resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_antiddos_lts_config" "test" {
  lts_group_id          = huaweicloud_lts_group.test.id
  lts_attack_stream_id  = huaweicloud_lts_stream.test.id
  enterprise_project_id = var.enterprise_project_id
}
```

**Parameter Description**:
- **lts_group_id**: The LTS log group ID, referencing the ID of the previously created Log Tank Service log group resource (huaweicloud_lts_group.test)
- **lts_attack_stream_id**: The LTS attack log stream ID, referencing the ID of the previously created Log Tank Service log stream resource (huaweicloud_lts_stream.test)
- **enterprise_project_id**: The enterprise project ID, assigned by referencing the input variable enterprise_project_id, default value is null

### 5. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Log Tank Service Log Group Configuration
lts_group_name  = "test-lts-group-name"
lts_ttl_in_days = 7

# Log Tank Service Log Stream Configuration
lts_stream_name = "test-lts-stream-name"
lts_is_favorite = true

# Enterprise Project Configuration
enterprise_project_id = "0"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="lts_group_name=test-group" -var="lts_stream_name=test-stream"`
2. Environment variables: `export TF_VAR_lts_group_name=test-group`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 6. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the Anti-DDoS LTS configuration
4. Run `terraform show` to view the details of the created Anti-DDoS LTS configuration

## Reference Information

- [Huawei Cloud Anti-DDoS Product Documentation](https://support.huaweicloud.com/antiddos/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Anti-DDoS LTS Configuration](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/antiddos/lts-config)
