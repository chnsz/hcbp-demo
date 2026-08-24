# Deploy ACL Rule Configuration

## Application Scenario

Cloud Firewall (CFW) is a new-generation cloud-native firewall that provides protection for internet boundaries and VPC boundaries on the cloud, including real-time intrusion detection and prevention, global unified access control, full traffic analysis visualization, and log auditing and traceability analysis. CFW supports ACL access control policies based on five-tuple, domain names, applications, IP address groups, and service groups to achieve fine-grained control over north-south and east-west traffic.

This best practice introduces how to use Terraform to automatically deploy CFW ACL rule configuration, including querying existing firewall information, creating IP address groups, service groups, domain name groups, and multiple types of ACL access control rules based on IP addresses, domain names, and address/service groups.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [CFW Firewalls Query Data Source (data.huaweicloud_cfw_firewalls)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/cfw_firewalls)

### Resources

- [CFW IP Address Group Resource (huaweicloud_cfw_address_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cfw_address_group)
- [CFW Service Group Resource (huaweicloud_cfw_service_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cfw_service_group)
- [CFW Domain Name Group Resource (huaweicloud_cfw_domain_name_group)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cfw_domain_name_group)
- [CFW ACL Rule Resource (huaweicloud_cfw_acl_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cfw_acl_rule)

### Resource/Data Source Dependencies

```
data.huaweicloud_cfw_firewalls
    ├── huaweicloud_cfw_address_group
    ├── huaweicloud_cfw_service_group
    ├── huaweicloud_cfw_domain_name_group
    └── huaweicloud_cfw_acl_rule.ip_based

huaweicloud_cfw_address_group
    └── huaweicloud_cfw_acl_rule.group_based

huaweicloud_cfw_service_group
    └── huaweicloud_cfw_acl_rule.group_based

huaweicloud_cfw_acl_rule.ip_based
    └── huaweicloud_cfw_acl_rule.domain_based
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) document for configuration introduction.

### 2. Query Firewall Information Through Data Source

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to perform a data source query, whose results are used to create CFW address groups, service groups, domain name groups, and ACL rules:

```hcl
variable "fw_instance_id" {
  description = "The firewall instance ID"
  type        = string
  default     = ""
  nullable    = false
}

# Query firewall information in the specified region (defaults to the region specified in the provider block when region parameter is omitted) for creating CFW address groups, service groups, domain name groups, and ACL rules
data "huaweicloud_cfw_firewalls" "test" {
  fw_instance_id = var.fw_instance_id != "" ? var.fw_instance_id : null
}

locals {
  fw_instance_id = var.fw_instance_id != "" ? var.fw_instance_id : try(data.huaweicloud_cfw_firewalls.test.records[0].fw_instance_id, null)
  object_id      = try(data.huaweicloud_cfw_firewalls.test.records[0].protect_objects[0].object_id, null)
}
```

**Parameter Description**:
- **fw_instance_id**: The firewall instance ID, assigned by referencing the input variable fw_instance_id, set to null when the value is an empty string to query the default firewall

### 3. Create IP Address Group

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a CFW IP address group resource:

```hcl
variable "address_group_name" {
  description = "The name of the IP address group"
  type        = string
}

variable "address_group_description" {
  description = "The description of the IP address group"
  type        = string
  default     = ""
}

# Create CFW IP address group resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_cfw_address_group" "test" {
  object_id   = local.object_id
  name        = var.address_group_name
  description = var.address_group_description
}
```

**Parameter Description**:
- **object_id**: The protected object ID, assigned based on the protect_objects returned by the firewall list query data source (data.huaweicloud_cfw_firewalls)
- **name**: The IP address group name, assigned by referencing the input variable address_group_name
- **description**: The IP address group description, assigned by referencing the input variable address_group_description, defaults to an empty string

### 4. Create Service Group

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a CFW service group resource:

```hcl
variable "service_group_name" {
  description = "The name of the service group"
  type        = string
}

variable "service_group_description" {
  description = "The description of the service group"
  type        = string
  default     = ""
}

# Create CFW service group resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_cfw_service_group" "test" {
  object_id   = local.object_id
  name        = var.service_group_name
  description = var.service_group_description
}
```

**Parameter Description**:
- **object_id**: The protected object ID, assigned based on the protect_objects returned by the firewall list query data source (data.huaweicloud_cfw_firewalls)
- **name**: The service group name, assigned by referencing the input variable service_group_name
- **description**: The service group description, assigned by referencing the input variable service_group_description, defaults to an empty string

### 5. Create Domain Name Group

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a CFW domain name group resource:

```hcl
variable "domain_name_group_name" {
  description = "The name of the domain name group"
  type        = string
}

variable "domain_name_group_type" {
  description = "The type of the domain name group"
  type        = number
  default     = 0
}

variable "domain_name_group_description" {
  description = "The description of the domain name group"
  type        = string
  default     = ""
}

variable "domain_name_group_domains" {
  description = "The list of domain names in the domain name group"
  type = list(object({
    domain_name = string
    description = string
  }))
  default = [
    {
      domain_name = "*.example.com"
      description = ""
    }
  ]
  nullable = false
}

# Create CFW domain name group resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_cfw_domain_name_group" "test" {
  fw_instance_id = local.fw_instance_id
  object_id      = local.object_id
  name           = var.domain_name_group_name
  type           = var.domain_name_group_type
  description    = var.domain_name_group_description

  dynamic "domain_names" {
    for_each = var.domain_name_group_domains

    content {
      domain_name = domain_names.value.domain_name
      description = domain_names.value.description
    }
  }
}
```

**Parameter Description**:
- **fw_instance_id**: The firewall instance ID, assigned based on the results returned by the firewall list query data source (data.huaweicloud_cfw_firewalls)
- **object_id**: The protected object ID, assigned based on the protect_objects returned by the firewall list query data source (data.huaweicloud_cfw_firewalls)
- **name**: The domain name group name, assigned by referencing the input variable domain_name_group_name
- **type**: The domain name group type, assigned by referencing the input variable domain_name_group_type, defaults to 0
- **description**: The domain name group description, assigned by referencing the input variable domain_name_group_description, defaults to an empty string
- **domain_names**: The domain name list, created through the dynamic block `dynamic "domain_names"` based on the input variable domain_name_group_domains
  - **domain_name**: The domain name, assigned by referencing the domain_name in the input variable
  - **description**: The domain name description, assigned by referencing the description in the input variable

### 6. Create IP-Based ACL Rule

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an IP-based CFW ACL rule resource:

```hcl
variable "acl_rule_ip_name" {
  description = "The name of the IP-based ACL rule"
  type        = string
}

variable "acl_rule_ip_description" {
  description = "The description of the IP-based ACL rule"
  type        = string
  default     = ""
}

variable "acl_rule_type" {
  description = "The ACL rule type. 0: Internet rule, 1: VPC rule, 2: NAT rule"
  type        = number
  default     = 0
}

variable "acl_rule_address_type" {
  description = "The ACL rule address type. 0: IPv4, 1: IPv6"
  type        = number
  default     = 0
}

variable "acl_rule_action_type" {
  description = "The ACL rule action type. 0: Allow, 1: Deny"
  type        = number
  default     = 0
}

variable "acl_rule_long_connect_enable" {
  description = "Whether to enable persistent connections. 0: disable, 1: enable"
  type        = number
  default     = 0
}

variable "acl_rule_status" {
  description = "The ACL rule status. 0: disable, 1: enable"
  type        = number
  default     = 1
}

variable "acl_rule_applications" {
  description = "The application list of the ACL rule"
  type        = list(string)
  default     = ["HTTPS"]
  nullable    = false
}

variable "acl_rule_source_addresses" {
  description = "The source IP address list of the ACL rule"
  type        = list(string)
  default     = ["1.1.1.1"]
  nullable    = false
}

variable "acl_rule_destination_addresses" {
  description = "The destination IP address list of the ACL rule"
  type        = list(string)
  default     = ["1.1.1.2"]
  nullable    = false
}

variable "acl_rule_custom_service_protocol" {
  description = "The protocol type of the custom service. 6: TCP, 17: UDP"
  type        = number
  default     = 6
}

variable "acl_rule_custom_service_source_port" {
  description = "The source port of the custom service"
  type        = string
  default     = "81"
}

variable "acl_rule_custom_service_dest_port" {
  description = "The destination port of the custom service"
  type        = string
  default     = "82"
}

variable "tags" {
  description = "The key/value pairs to associate with the resources"
  type        = map(string)
  default = {
    key = "value"
  }
  nullable = false
}

# Create IP-based CFW ACL rule resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_cfw_acl_rule" "ip_based" {
  name                = var.acl_rule_ip_name
  object_id           = local.object_id
  description         = var.acl_rule_ip_description
  type                = var.acl_rule_type
  address_type        = var.acl_rule_address_type
  action_type         = var.acl_rule_action_type
  long_connect_enable = var.acl_rule_long_connect_enable
  status              = var.acl_rule_status
  applications        = var.acl_rule_applications

  source_addresses      = var.acl_rule_source_addresses
  destination_addresses = var.acl_rule_destination_addresses

  custom_services {
    protocol    = var.acl_rule_custom_service_protocol
    source_port = var.acl_rule_custom_service_source_port
    dest_port   = var.acl_rule_custom_service_dest_port
  }

  sequence {
    top = 1
  }

  tags = var.tags
}
```

**Parameter Description**:
- **name**: The ACL rule name, assigned by referencing the input variable acl_rule_ip_name
- **object_id**: The protected object ID, assigned based on the protect_objects returned by the firewall list query data source (data.huaweicloud_cfw_firewalls)
- **description**: The ACL rule description, assigned by referencing the input variable acl_rule_ip_description, defaults to an empty string
- **type**: The ACL rule type, assigned by referencing the input variable acl_rule_type, 0 indicates internet rule, 1 indicates VPC rule, 2 indicates NAT rule, defaults to 0
- **address_type**: The ACL rule address type, assigned by referencing the input variable acl_rule_address_type, 0 indicates IPv4, 1 indicates IPv6, defaults to 0
- **action_type**: The ACL rule action type, assigned by referencing the input variable acl_rule_action_type, 0 indicates allow, 1 indicates deny, defaults to 0
- **long_connect_enable**: Whether to enable persistent connections, assigned by referencing the input variable acl_rule_long_connect_enable, 0 indicates disable, 1 indicates enable, defaults to 0
- **status**: The ACL rule status, assigned by referencing the input variable acl_rule_status, 0 indicates disable, 1 indicates enable, defaults to 1
- **applications**: The ACL rule application list, assigned by referencing the input variable acl_rule_applications, defaults to ["HTTPS"]
- **source_addresses**: The source IP address list, assigned by referencing the input variable acl_rule_source_addresses, defaults to ["1.1.1.1"]
- **destination_addresses**: The destination IP address list, assigned by referencing the input variable acl_rule_destination_addresses, defaults to ["1.1.1.2"]
- **custom_services**: The custom service configuration block
  - **protocol**: The custom service protocol type, assigned by referencing the input variable acl_rule_custom_service_protocol, 6 indicates TCP, 17 indicates UDP, defaults to 6
  - **source_port**: The custom service source port, assigned by referencing the input variable acl_rule_custom_service_source_port, defaults to "81"
  - **dest_port**: The custom service destination port, assigned by referencing the input variable acl_rule_custom_service_dest_port, defaults to "82"
- **sequence**: The rule priority configuration block
  - **top**: The top priority of the rule, set to 1
- **tags**: The key/value pairs associated with the resources, assigned by referencing the input variable tags

### 7. Create Domain-Based ACL Rule

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a domain-based CFW ACL rule resource:

```hcl
variable "acl_rule_domain_name" {
  description = "The name of the domain-based ACL rule"
  type        = string
}

variable "acl_rule_domain_description" {
  description = "The description of the domain-based ACL rule"
  type        = string
  default     = ""
}

variable "acl_rule_domain_direction" {
  description = "The direction of the domain-based ACL rule. 0: inbound, 1: outbound"
  type        = number
  default     = 1
}

variable "acl_rule_destination_domain_address_name" {
  description = "The destination domain address name"
  type        = string
  default     = "*.baidu.com"
}

# Create domain-based CFW ACL rule resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_cfw_acl_rule" "domain_based" {
  name                = var.acl_rule_domain_name
  object_id           = local.object_id
  description         = var.acl_rule_domain_description
  type                = var.acl_rule_type
  address_type        = var.acl_rule_address_type
  action_type         = var.acl_rule_action_type
  long_connect_enable = var.acl_rule_long_connect_enable
  status              = var.acl_rule_status
  direction           = var.acl_rule_domain_direction

  source_addresses                = var.acl_rule_source_addresses
  destination_domain_address_name = var.acl_rule_destination_domain_address_name

  custom_services {
    protocol    = var.acl_rule_custom_service_protocol
    source_port = var.acl_rule_custom_service_source_port
    dest_port   = var.acl_rule_custom_service_dest_port
  }

  sequence {
    top          = 0
    dest_rule_id = huaweicloud_cfw_acl_rule.ip_based.id
  }

  tags = var.tags
}
```

**Parameter Description**:
- **name**: The ACL rule name, assigned by referencing the input variable acl_rule_domain_name
- **object_id**: The protected object ID, assigned based on the protect_objects returned by the firewall list query data source (data.huaweicloud_cfw_firewalls)
- **description**: The ACL rule description, assigned by referencing the input variable acl_rule_domain_description, defaults to an empty string
- **type**: The ACL rule type, assigned by referencing the input variable acl_rule_type
- **address_type**: The ACL rule address type, assigned by referencing the input variable acl_rule_address_type
- **action_type**: The ACL rule action type, assigned by referencing the input variable acl_rule_action_type
- **long_connect_enable**: Whether to enable persistent connections, assigned by referencing the input variable acl_rule_long_connect_enable
- **status**: The ACL rule status, assigned by referencing the input variable acl_rule_status
- **direction**: The ACL rule direction, assigned by referencing the input variable acl_rule_domain_direction, 0 indicates inbound, 1 indicates outbound, defaults to 1
- **source_addresses**: The source IP address list, assigned by referencing the input variable acl_rule_source_addresses
- **destination_domain_address_name**: The destination domain address name, assigned by referencing the input variable acl_rule_destination_domain_address_name, defaults to "*.baidu.com"
- **custom_services**: The custom service configuration block
  - **protocol**: The custom service protocol type, assigned by referencing the input variable acl_rule_custom_service_protocol
  - **source_port**: The custom service source port, assigned by referencing the input variable acl_rule_custom_service_source_port
  - **dest_port**: The custom service destination port, assigned by referencing the input variable acl_rule_custom_service_dest_port
- **sequence**: The rule priority configuration block
  - **top**: The top priority of the rule, set to 0
  - **dest_rule_id**: The destination rule ID, referencing the ID of the previously created IP-based ACL rule resource (huaweicloud_cfw_acl_rule.ip_based)
- **tags**: The key/value pairs associated with the resources, assigned by referencing the input variable tags

### 8. Create Address Group and Service Group Based ACL Rule

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an address group and service group based CFW ACL rule resource:

```hcl
variable "acl_rule_group_name" {
  description = "The name of the group-based ACL rule"
  type        = string
}

variable "acl_rule_group_description" {
  description = "The description of the group-based ACL rule"
  type        = string
  default     = ""
}

variable "acl_rule_service_group_protocol" {
  description = "The protocol type used by the service group"
  type        = number
  default     = 6
}

# Create address group and service group based CFW ACL rule resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_cfw_acl_rule" "group_based" {
  name                = var.acl_rule_group_name
  object_id           = local.object_id
  description         = var.acl_rule_group_description
  type                = var.acl_rule_type
  address_type        = var.acl_rule_address_type
  action_type         = var.acl_rule_action_type
  long_connect_enable = var.acl_rule_long_connect_enable
  status              = var.acl_rule_status

  source_address_groups      = [huaweicloud_cfw_address_group.test.id]
  destination_address_groups = [huaweicloud_cfw_address_group.test.id]

  custom_service_groups {
    protocols = [var.acl_rule_service_group_protocol]
    group_ids = [huaweicloud_cfw_service_group.test.id]
  }

  sequence {
    bottom = 1
  }

  tags = var.tags
}
```

**Parameter Description**:
- **name**: The ACL rule name, assigned by referencing the input variable acl_rule_group_name
- **object_id**: The protected object ID, assigned based on the protect_objects returned by the firewall list query data source (data.huaweicloud_cfw_firewalls)
- **description**: The ACL rule description, assigned by referencing the input variable acl_rule_group_description, defaults to an empty string
- **type**: The ACL rule type, assigned by referencing the input variable acl_rule_type
- **address_type**: The ACL rule address type, assigned by referencing the input variable acl_rule_address_type
- **action_type**: The ACL rule action type, assigned by referencing the input variable acl_rule_action_type
- **long_connect_enable**: Whether to enable persistent connections, assigned by referencing the input variable acl_rule_long_connect_enable
- **status**: The ACL rule status, assigned by referencing the input variable acl_rule_status
- **source_address_groups**: The source IP address group ID list, referencing the ID of the previously created CFW IP address group resource (huaweicloud_cfw_address_group.test)
- **destination_address_groups**: The destination IP address group ID list, referencing the ID of the previously created CFW IP address group resource (huaweicloud_cfw_address_group.test)
- **custom_service_groups**: The custom service group configuration block
  - **protocols**: The service group protocol type list, assigned by referencing the input variable acl_rule_service_group_protocol, defaults to 6 (TCP)
  - **group_ids**: The service group ID list, referencing the ID of the previously created CFW service group resource (huaweicloud_cfw_service_group.test)
- **sequence**: The rule priority configuration block
  - **bottom**: The bottom priority of the rule, set to 1
- **tags**: The key/value pairs associated with the resources, assigned by referencing the input variable tags

### 9. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Firewall instance
fw_instance_id = "your_firewall_instance_id"

# IP address group configuration
address_group_name = "tf_test_address_group"

# Service group configuration
service_group_name = "tf_test_service_group"

# Domain name group configuration
domain_name_group_name = "tf_test_domain_name_group"

# ACL rule configuration
acl_rule_ip_name     = "tf_test_acl_rule_ip"
acl_rule_domain_name = "tf_test_acl_rule_domain"
acl_rule_group_name  = "tf_test_acl_rule_group"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="fw_instance_id=your-firewall-id" -var="acl_rule_ip_name=tf_test_acl_rule_ip"`
2. Environment variables: `export TF_VAR_acl_rule_ip_name=tf_test_acl_rule_ip`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 10. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the CFW ACL rule configuration
4. Run `terraform show` to view the created CFW ACL rule configuration

## Reference Information

- [Huawei Cloud CFW Product Documentation](https://support.huaweicloud.com/cfw/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For CFW ACL Rule Configuration](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cfw/acl-rule-config)
