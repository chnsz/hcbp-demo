# Deploy Quota

## Application Scenario

Host Security Service (HSS) is a host security protection service provided by Huawei Cloud, offering asset management, vulnerability management, intrusion detection, baseline checks, and other functions to help you comprehensively protect the security of cloud hosts. By purchasing HSS quotas, you can provide prepaid security protection services for cloud hosts, including real-time monitoring, threat detection, vulnerability scanning, baseline checks, and other functions, ensuring host security. This best practice will introduce how to use Terraform to automatically deploy HSS quotas, purchase prepaid HSS protection quotas, and provide continuous security protection capabilities for cloud hosts.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [HSS Quota Resource (huaweicloud_hss_quota)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/hss_quota)

### Resource/Data Source Dependencies

```text
huaweicloud_hss_quota
```

> Note: HSS quota resource is an independent resource used to purchase HSS protection quotas. After purchasing quotas, you can assign quotas to hosts that need protection.

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create HSS Quota Resource

Add the following script to the TF file (e.g., main.tf) to create HSS quota:

```hcl
variable "quota_version" {
  description = "The protection quota version"
  type        = string
}

variable "period_unit" {
  description = "The charging period unit of the quota"
  type        = string
}

variable "period" {
  description = "The charging period of the quota"
  type        = number
}

variable "is_auto_renew" {
  description = "Whether auto-renew is enabled"
  type        = bool
  default     = false
}

variable "enterprise_project_id" {
  description = "The enterprise project ID to which the HSS quota belongs"
  type        = string
  default     = null
}

variable "quota_tags" {
  description = "The key/value pairs to associate with the HSS quota"
  type        = map(string)
  default     = null
}

# Create HSS quota resource in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block)
resource "huaweicloud_hss_quota" "test" {
  version               = var.quota_version
  period_unit           = var.period_unit
  period                = var.period
  auto_renew            = var.is_auto_renew
  enterprise_project_id = var.enterprise_project_id
  tags                  = var.quota_tags
}
```

**Parameter Description**:
- **version**: Protection quota version, assigned by referencing input variable quota_version, for example "hss.version.enterprise" represents enterprise edition
- **period_unit**: Charge period unit, assigned by referencing input variable period_unit, optional values are "month" or "year"
- **period**: Charge period, assigned by referencing input variable period, represents the purchase duration, for example 1 represents 1 month or 1 year
- **auto_renew**: Whether auto-renewal is enabled, assigned by referencing input variable is_auto_renew, default value is false
- **enterprise_project_id**: Enterprise project ID, assigned by referencing input variable enterprise_project_id, optional parameter, default value is null
- **tags**: Quota tags, assigned by referencing input variable quota_tags, optional parameter, default value is null

> Note: HSS quota uses prepaid billing mode, requiring specification of charge period unit and period. After purchasing quotas, you can assign quotas to hosts that need protection. The quota version needs to be selected according to actual requirements, common versions include basic edition, enterprise edition, etc.

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# HSS Quota Configuration
quota_version         = "hss.version.enterprise"
period_unit           = "month"
period                = 1
is_auto_renew         = false
enterprise_project_id = "0"
quota_tags            = {
  foo = "bar"
  key = "value"
}
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs, especially:
   - `quota_version` can be set to "hss.version.enterprise" (enterprise edition) or other supported quota versions
   - `period_unit` can be set to "month" or "year"
   - `period` can be set to the purchase duration, for example 1 represents 1 month or 1 year
   - `is_auto_renew` can be set to true to enable auto-renewal
   - `enterprise_project_id` can be set to enterprise project ID, can be omitted or set to "0" if not needed
   - `quota_tags` can be set to quota tags for resource classification and management, can be omitted if not needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="quota_version=hss.version.enterprise" -var="period_unit=month" -var="period=1"`
2. Environment variables: `export TF_VAR_quota_version=hss.version.enterprise` and `export TF_VAR_period_unit=month`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values. HSS quota uses prepaid billing mode, and charges will be incurred after purchase. Please select the appropriate quota version and purchase duration according to actual requirements.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create HSS quota:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating HSS quota
4. Run `terraform show` to view the details of the created HSS quota

> Note: Charges will be incurred after HSS quota creation. Please ensure sufficient account balance. After purchasing quotas, you can assign quotas to hosts that need protection. If auto-renewal is enabled, quotas will be automatically renewed before expiration.

## Reference Information

- [Huawei Cloud HSS Product Documentation](https://support.huaweicloud.com/hss/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Quota](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/hss/prepaid-quota)
