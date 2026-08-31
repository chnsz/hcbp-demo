# Deploy Black and White List

## Application Scenario

Cloud Firewall (CFW) is a new-generation cloud-native firewall that provides protection for internet boundaries and VPC boundaries on the cloud, including real-time intrusion detection and prevention, global unified access control, full traffic analysis visualization, and log auditing and traceability analysis. CFW supports access control for specified IP addresses, ports, and protocols through blacklists and whitelists. Blacklists are used to block malicious traffic, and whitelists are used to allow trusted traffic.

This best practice introduces how to use Terraform to automatically deploy CFW black and white list rules, including querying existing firewall information and creating blacklist and whitelist rules to block and allow specified IP addresses respectively.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [CFW Firewalls Query Data Source (data.huaweicloud_cfw_firewalls)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/cfw_firewalls)

### Resources

- [CFW Black and White List Resource (huaweicloud_cfw_black_white_list)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cfw_black_white_list)

### Resource/Data Source Dependencies

```
data.huaweicloud_cfw_firewalls
    └── huaweicloud_cfw_black_white_list
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) document for configuration introduction.

### 2. Query Firewall Information Through Data Source

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, whose results are used to create CFW black and white list rules:

```hcl
variable "fw_instance_id" {
  description = "The firewall instance ID"
  type        = string
  default     = ""
  nullable    = false
}

# Query firewall information in the specified region (defaults to the region specified in the provider block when region parameter is omitted) for creating CFW black and white list rules
data "huaweicloud_cfw_firewalls" "test" {
  fw_instance_id = var.fw_instance_id != "" ? var.fw_instance_id : null
}

locals {
  object_id = try(data.huaweicloud_cfw_firewalls.test.records[0].protect_objects[0].object_id, null)
}
```

**Parameter Description**:
- **fw_instance_id**: The firewall instance ID, assigned by referencing the input variable fw_instance_id, set to null when the value is an empty string to query the default firewall

### 3. Create Black and White List Rules

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create CFW black and white list rule resources:

```hcl
variable "blacklist_list_type" {
  description = "The list type of the blacklist rule. 4: blacklist, 5: whitelist"
  type        = number
  default     = 4
}

variable "whitelist_list_type" {
  description = "The list type of the whitelist rule. 4: blacklist, 5: whitelist"
  type        = number
  default     = 5
}

variable "blacklist_direction" {
  description = "The direction of the blacklist rule. 0: inbound, 1: outbound"
  type        = number
  default     = 0
}

variable "whitelist_direction" {
  description = "The direction of the whitelist rule. 0: inbound, 1: outbound"
  type        = number
  default     = 0
}

variable "blacklist_protocol" {
  description = "The protocol type of the blacklist rule. 6: TCP, 17: UDP, -1: any"
  type        = number
  default     = 6
}

variable "whitelist_protocol" {
  description = "The protocol type of the whitelist rule. 6: TCP, 17: UDP, -1: any"
  type        = number
  default     = 6
}

variable "blacklist_port" {
  description = "The destination port of the blacklist rule"
  type        = string
  default     = "22"
}

variable "whitelist_port" {
  description = "The destination port of the whitelist rule"
  type        = string
  default     = "80"
}

variable "blacklist_address_type" {
  description = "The IP address type of the blacklist rule. 0: IPv4, 1: IPv6"
  type        = number
  default     = 0
}

variable "whitelist_address_type" {
  description = "The IP address type of the whitelist rule. 0: IPv4, 1: IPv6"
  type        = number
  default     = 0
}

variable "blacklist_address" {
  description = "The IP address of the blacklist rule"
  type        = string
}

variable "whitelist_address" {
  description = "The IP address of the whitelist rule"
  type        = string
}

# Create CFW black and white list rule resources in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_cfw_black_white_list" "test" {
  count = 2

  object_id    = local.object_id
  list_type    = count.index == 0 ? var.blacklist_list_type : var.whitelist_list_type
  direction    = count.index == 0 ? var.blacklist_direction : var.whitelist_direction
  protocol     = count.index == 0 ? var.blacklist_protocol : var.whitelist_protocol
  port         = count.index == 0 ? var.blacklist_port : var.whitelist_port
  address_type = count.index == 0 ? var.blacklist_address_type : var.whitelist_address_type
  address      = count.index == 0 ? var.blacklist_address : var.whitelist_address
}
```

**Parameter Description**:
- **count**: The number of black and white list resources to create, set to 2 to create one blacklist rule and one whitelist rule respectively
- **object_id**: The protected object ID, assigned based on the protect_objects returned by the firewall list query data source (data.huaweicloud_cfw_firewalls)
- **list_type**: The list type, uses the blacklist list type variable blacklist_list_type (defaults to 4) when count.index is 0, otherwise uses the whitelist list type variable whitelist_list_type (defaults to 5)
- **direction**: The rule direction, uses the blacklist direction variable blacklist_direction (defaults to 0, inbound) when count.index is 0, otherwise uses the whitelist direction variable whitelist_direction (defaults to 0)
- **protocol**: The protocol type, uses the blacklist protocol variable blacklist_protocol (defaults to 6, TCP) when count.index is 0, otherwise uses the whitelist protocol variable whitelist_protocol (defaults to 6)
- **port**: The destination port, uses the blacklist port variable blacklist_port (defaults to "22") when count.index is 0, otherwise uses the whitelist port variable whitelist_port (defaults to "80")
- **address_type**: The IP address type, uses the blacklist address type variable blacklist_address_type (defaults to 0, IPv4) when count.index is 0, otherwise uses the whitelist address type variable whitelist_address_type (defaults to 0)
- **address**: The IP address, assigned by referencing the input variable blacklist_address when count.index is 0, otherwise assigned by referencing the input variable whitelist_address

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Firewall instance
fw_instance_id = "your_firewall_instance_id"

# Blacklist rule configuration
blacklist_list_type    = 4
blacklist_direction    = 0
blacklist_protocol     = 6
blacklist_port         = "22"
blacklist_address_type = 0
blacklist_address      = "1.1.1.1"

# Whitelist rule configuration
whitelist_list_type    = 5
whitelist_direction    = 0
whitelist_protocol     = 6
whitelist_port         = "443"
whitelist_address_type = 0
whitelist_address      = "2.2.2.2"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="fw_instance_id=your-firewall-id" -var="blacklist_address=1.1.1.1"`
2. Environment variables: `export TF_VAR_blacklist_address=1.1.1.1`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the CFW black and white list rules
4. Run `terraform show` to view the created CFW black and white list rules

## Reference Information

- [Huawei Cloud CFW Product Documentation](https://support.huaweicloud.com/cfw/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For CFW Black and White List](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cfw/black-white-list)
