# Deploy Host Protection

## Application Scenario

Host Security Service (HSS) is a host security protection service provided by Huawei Cloud, offering asset management, vulnerability management, intrusion detection, baseline checks, and other functions to help you comprehensively protect the security of cloud hosts. By enabling HSS protection for existing hosts, you can provide comprehensive security protection capabilities for cloud hosts, including real-time monitoring, threat detection, vulnerability scanning, baseline checks, and other functions, ensuring host security. This best practice will introduce how to use Terraform to automatically deploy HSS host protection, enabling pay-per-use HSS protection services for existing hosts.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [HSS Host Protection Resource (huaweicloud_hss_host_protection)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/hss_host_protection)

### Resource/Data Source Dependencies

```text
huaweicloud_hss_host_protection
```

> Note: HSS host protection resource depends on an existing host (ECS instance), which needs to have HSS agent installed. The host ID is specified through input variables.

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create HSS Host Protection Resource

Add the following script to the TF file (e.g., main.tf) to create HSS host protection:

```hcl
variable "host_id" {
  description = "The host ID for the host protection"
  type        = string
}

variable "protection_version" {
  description = "The protection version enabled by the host"
  type        = string
}

variable "is_wait_host_available" {
  description = "Whether to wait for the host agent status to become online"
  type        = bool
  default     = false
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project to which the host protection belongs"
  type        = string
  default     = null
}

# Create HSS host protection resource in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block)
resource "huaweicloud_hss_host_protection" "test" {
  host_id                = var.host_id
  version                = var.protection_version
  charging_mode          = "postPaid"
  is_wait_host_available = var.is_wait_host_available
  enterprise_project_id  = var.enterprise_project_id
}
```

**Parameter Description**:
- **host_id**: Host ID, assigned by referencing input variable host_id, must be an existing ECS instance ID with HSS agent installed
- **version**: Protection version, assigned by referencing input variable protection_version, for example "hss.version.enterprise" represents enterprise edition
- **charging_mode**: Charge mode, set to "postPaid" to indicate pay-per-use
- **is_wait_host_available**: Whether to wait for the host agent status to become online, assigned by referencing input variable is_wait_host_available, default value is false
- **enterprise_project_id**: Enterprise project ID, assigned by referencing input variable enterprise_project_id, optional parameter, default value is null

> Note: HSS host protection depends on an existing host, which must have HSS agent installed. If the host has not installed HSS agent, you need to install HSS agent for the ECS instance first (can be achieved by configuring `agent_list = "hss"` when creating the ECS instance). The protection version parameter needs to be selected according to actual requirements, common versions include basic edition, enterprise edition, etc.

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# HSS Host Protection Configuration
host_id            = "your_host_id"
protection_version = "hss.version.enterprise"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs, especially:
   - `host_id` needs to be set to an existing ECS instance ID with HSS agent installed
   - `protection_version` can be set to "hss.version.enterprise" (enterprise edition) or other supported protection versions
   - `is_wait_host_available` can be set to true, indicating to wait for the host agent status to become online before creating protection
   - `enterprise_project_id` can be set to enterprise project ID, can be omitted if not needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="host_id=your_host_id" -var="protection_version=hss.version.enterprise"`
2. Environment variables: `export TF_VAR_host_id=your_host_id` and `export TF_VAR_protection_version=hss.version.enterprise`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values. Ensure that the ECS instance corresponding to the specified host ID has HSS agent installed, otherwise protection creation may fail.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create HSS host protection:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating HSS host protection
4. Run `terraform show` to view the details of the created HSS host protection

> Note: Before creating HSS host protection, ensure that the specified host has HSS agent installed. If the host has not installed HSS agent, you need to install HSS agent for the ECS instance first. If `is_wait_host_available = true` is set, Terraform will wait for the host agent status to become online before creating protection, which may take some time.

## Reference Information

- [Huawei Cloud HSS Product Documentation](https://support.huaweicloud.com/hss/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Host Protection](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/hss/postpaid-host-protection)
