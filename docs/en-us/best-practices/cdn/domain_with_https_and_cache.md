# Deploy HTTPS and Cache Domain

## Application Scenario

Content Delivery Network (CDN) domain is a domain acceleration configuration function provided by the CDN service, used to provide content acceleration services for websites, downloads, videos and other businesses. By configuring a CDN domain, you can distribute origin server content to global edge nodes, improving user access speed. By configuring HTTPS, you can ensure the security of data transmission. By configuring cache rules, you can optimize cache policies and improve cache hit rates. Automating CDN domain creation through Terraform can ensure standardized and consistent domain configuration, improving operational efficiency. This best practice will introduce how to use Terraform to automatically create a CDN domain, including HTTPS and cache rule configuration.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [CDN Domain Resource (huaweicloud_cdn_domain)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cdn_domain)

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create CDN Domain Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a CDN domain resource:

```hcl
variable "domain_name" {
  description = "The name of the CDN domain to be accelerated"
  type        = string
}

variable "domain_type" {
  description = "The business type of the domain"
  type        = string
  default     = "web"

  validation {
    condition     = contains(["web", "download", "video", "wholeSite"], var.domain_type)
    error_message = "The domain_type must be one of: web, download, video, wholeSite."
  }
}

variable "service_area" {
  description = "The area covered by the acceleration service"
  type        = string
  default     = "mainland_china"

  validation {
    condition     = contains(["mainland_china", "outside_mainland_china", "global"], var.service_area)
    error_message = "The service_area must be one of: mainland_china, outside_mainland_china, global."
  }
}

variable "origin_server" {
  description = "The origin server address (IP address or domain name)"
  type        = string
}

variable "origin_type" {
  description = "The origin server type"
  type        = string
  default     = "ipaddr"

  validation {
    condition     = contains(["ipaddr", "domain", "obs_bucket"], var.origin_type)
    error_message = "The origin_type must be one of: ipaddr, domain, obs_bucket."
  }
}

variable "http_port" {
  description = "The HTTP port of the origin server"
  type        = number
  default     = 80
}

variable "https_port" {
  description = "The HTTPS port of the origin server"
  type        = number
  default     = 443
}

variable "origin_protocol" {
  description = "The protocol used to retrieve data from the origin server"
  type        = string
  default     = "http"

  validation {
    condition     = contains(["http", "https", "follow"], var.origin_protocol)
    error_message = "The origin_protocol must be one of: http, https, follow."
  }
}

variable "ipv6_enable" {
  description = "Whether to enable IPv6"
  type        = bool
  default     = false
}

variable "range_based_retrieval_enabled" {
  description = "Whether to enable range-based retrieval"
  type        = bool
  default     = false
}

variable "domain_description" {
  description = "The description of the CDN domain"
  type        = string
  default     = ""
}

variable "https_enabled" {
  description = "Whether to enable HTTPS"
  type        = bool
  default     = false
}

variable "certificate_name" {
  description = "The name of the SSL certificate (required when https_enabled is true)"
  type        = string
  default     = ""
  nullable    = false
}

variable "certificate_source" {
  description = "The source of the SSL certificate (required when https_enabled is true)"
  type        = string
  default     = "0"
  nullable    = false

  validation {
    condition     = contains(["0", "2"], var.certificate_source)
    error_message = "The certificate_source must be one of: 0, 2."
  }
}

variable "certificate_body_path" {
  description = "The file path to the SSL certificate (required when https_enabled is true and using custom certificate)"
  type        = string
  default     = ""
  sensitive   = false
  nullable    = false
}

variable "private_key_path" {
  description = "The file path to the private key (required when https_enabled is true and using custom certificate)"
  type        = string
  default     = ""
  sensitive   = false
  nullable    = false
}

variable "http2_enabled" {
  description = "Whether to enable HTTP/2 (only valid when https_enabled is true)"
  type        = bool
  default     = false
}

variable "ocsp_stapling_status" {
  description = "The OCSP stapling status (only valid when https_enabled is true)"
  type        = string
  default     = "off"

  validation {
    condition     = contains(["on", "off"], var.ocsp_stapling_status)
    error_message = "The ocsp_stapling_status must be one of: on, off."
  }
}

variable "cache_rules" {
  description = "The cache rules configuration"
  type        = list(object({
    rule_type           = string
    content             = string
    ttl                 = number
    ttl_type            = string
    priority            = number
    url_parameter_type  = optional(string)
    url_parameter_value = optional(string)
  }))
  default     = []
}

variable "domain_tags" {
  description = "The tags of the CDN domain"
  type        = map(string)
  default     = {}
}

# Create CDN domain resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_cdn_domain" "test" {
  name         = var.domain_name
  type         = var.domain_type
  service_area = var.service_area

  sources {
    origin      = var.origin_server
    origin_type = var.origin_type
    active      = 1
    http_port   = var.http_port
    https_port  = var.https_port
  }

  configs {
    origin_protocol               = var.origin_protocol
    ipv6_enable                   = var.ipv6_enable
    range_based_retrieval_enabled = var.range_based_retrieval_enabled
    description                   = var.domain_description

    dynamic "https_settings" {
      for_each = var.https_enabled ? [1] : []

      content {
        certificate_name     = var.https_enabled ? var.certificate_name : null
        certificate_source   = var.https_enabled ? var.certificate_source : null
        certificate_body     = var.https_enabled && var.certificate_body_path != "" ? file(var.certificate_body_path) : null
        private_key          = var.https_enabled && var.private_key_path != "" ? file(var.private_key_path) : null
        https_enabled        = var.https_enabled
        http2_enabled        = var.http2_enabled
        ocsp_stapling_status = var.ocsp_stapling_status
      }
    }
  }

  dynamic "cache_settings" {
    for_each = length(var.cache_rules) > 0 ? [var.cache_rules] : []

    content {
      dynamic "rules" {
        for_each = cache_settings.value

        content {
          rule_type           = rules.value.rule_type
          ttl                 = rules.value.ttl
          ttl_type            = rules.value.ttl_type
          priority            = rules.value.priority
          content             = rules.value.content
          url_parameter_type  = lookup(rules.value, "url_parameter_type", null)
          url_parameter_value = lookup(rules.value, "url_parameter_value", null)
        }
      }
    }
  }

  tags = var.domain_tags
}
```

**Parameter Description**:
- **name**: The accelerated domain name, assigned by referencing the input variable domain_name
- **type**: The domain business type, assigned by referencing the input variable domain_type, valid values: web (website acceleration), download (file download acceleration), video (on-demand acceleration), wholeSite (full-site acceleration), default value is "web"
- **service_area**: The service coverage area, assigned by referencing the input variable service_area, valid values: mainland_china (mainland China), outside_mainland_china (outside mainland China), global (global), default value is "mainland_china"
- **sources.origin**: The origin server address, assigned by referencing the input variable origin_server, can be an IP address or domain name
- **sources.origin_type**: The origin server type, assigned by referencing the input variable origin_type, valid values: ipaddr (IP address), domain (domain name), obs_bucket (OBS bucket domain), default value is "ipaddr"
- **sources.active**: The primary/standby status, set to 1 for primary origin server
- **sources.http_port**: The origin server HTTP port, assigned by referencing the input variable http_port, default value is 80
- **sources.https_port**: The origin server HTTPS port, assigned by referencing the input variable https_port, default value is 443
- **configs.origin_protocol**: The origin protocol, assigned by referencing the input variable origin_protocol, valid values: http, https, follow (follow), default value is "http"
- **configs.ipv6_enable**: Whether to enable IPv6, assigned by referencing the input variable ipv6_enable, default value is false
- **configs.range_based_retrieval_enabled**: Whether to enable Range-based retrieval, assigned by referencing the input variable range_based_retrieval_enabled, default value is false
- **configs.description**: The domain description, assigned by referencing the input variable domain_description, default value is empty string
- **configs.https_settings.certificate_name**: The certificate name, assigned by referencing the input variable certificate_name when https_enabled is true
- **configs.https_settings.certificate_source**: The certificate source, assigned by referencing the input variable certificate_source when https_enabled is true, valid values: 0 (Huawei Cloud managed certificate), 2 (custom certificate), default value is "0"
- **configs.https_settings.certificate_body**: The certificate content, read from certificate_body_path file using file function when https_enabled is true and using custom certificate
- **configs.https_settings.private_key**: The private key content, read from private_key_path file using file function when https_enabled is true and using custom certificate
- **configs.https_settings.https_enabled**: Whether to enable HTTPS, assigned by referencing the input variable https_enabled, default value is false
- **configs.https_settings.http2_enabled**: Whether to enable HTTP/2, assigned by referencing the input variable http2_enabled, only valid when https_enabled is true, default value is false
- **configs.https_settings.ocsp_stapling_status**: The OCSP stapling status, assigned by referencing the input variable ocsp_stapling_status, only valid when https_enabled is true, valid values: on (enabled), off (disabled), default value is "off"
- **cache_settings.rules.rule_type**: The cache rule type, valid values: all (all files), file_extension (file extension), catalog (directory), full_path (full path), home_page (homepage)
- **cache_settings.rules.content**: The cache rule matching content, set different matching content according to rule_type
- **cache_settings.rules.ttl**: The cache time, in the unit specified by ttl_type, maximum cache time is 365 days
- **cache_settings.rules.ttl_type**: The cache time unit, valid values: s (second), m (minute), h (hour), d (day)
- **cache_settings.rules.priority**: The cache rule priority, larger value indicates higher priority, value range is 1-100, weight values must be unique
- **cache_settings.rules.url_parameter_type**: The URL parameter type, valid values: del_params (ignore specified URL parameters), reserve_params (retain specified URL parameters), ignore_url_params (ignore all URL parameters), full_url (retain all URL parameters), default value is "full_url"
- **cache_settings.rules.url_parameter_value**: The URL parameter values, multiple parameters separated by commas, up to 10 parameters can be set, required when url_parameter_type is del_params or reserve_params
- **tags**: The domain tags, assigned by referencing the input variable domain_tags, default value is empty map

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Domain Configuration
domain_name                   = "example.com"
domain_type                   = "web"
service_area                  = "outside_mainland_china"
origin_protocol               = "https"
origin_type                   = "domain"
origin_server                 = "hostaddress"
http_port                     = 80
https_port                    = 443
ipv6_enable                   = false
range_based_retrieval_enabled = false
domain_description            = "CDN domain for example.com"

# HTTPS Configuration
https_enabled                 = true
certificate_name              = "terraform_test_cert"
certificate_source            = "0"
certificate_body_path         = "/path/to/your/certificate.crt"
private_key_path              = "/path/to/your/private.key"
http2_enabled                 = true
ocsp_stapling_status          = "on"

# Cache Rules Configuration
cache_rules = [
  {
    rule_type          = "all"
    content            = ""
    ttl                = 2592000
    ttl_type           = "s"
    priority           = 1
    url_parameter_type = "full_url"
  },
  {
    rule_type          = "file_extension"
    content            = ".php;.jsp;.asp;.aspx"
    ttl                = 2592000
    ttl_type           = "s"
    priority           = 2
    url_parameter_type = "full_url"
  }
]

domain_tags = {
  Environment = "production"
  Project     = "cdn-example"
}
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="domain_name=example.com" -var="origin_server=192.168.1.100"`
2. Environment variables: `export TF_VAR_domain_name=example.com` and `export TF_VAR_origin_server=192.168.1.100`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create a CDN domain:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the CDN domain
4. Run `terraform show` to view the details of the created CDN domain

> Note: CDN domain creation may take a few minutes to complete. Before updating the domain configuration, please ensure that the status value is **online**. The service area cannot be changed between mainland China and outside mainland China. SSL certificate files should be kept secure and never committed to version control. Cache rules are processed in priority order (smaller number = higher priority). Domain names must be unique within your Huawei Cloud account.

## Reference Information

- [Huawei Cloud CDN Product Documentation](https://support.huaweicloud.com/cdn/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For HTTPS and Cache Domain](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cdn/domain-with-https-and-cache)
