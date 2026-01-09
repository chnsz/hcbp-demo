# Deploy Rule Engine

## Application Scenario

Content Delivery Network (CDN) rule engine is a flexible rule configuration function provided by the CDN service, used to execute corresponding actions based on different request conditions, achieving fine-grained CDN acceleration control. By configuring the rule engine, you can match requests based on conditions such as path, parameters, request headers, etc., and execute various actions such as cache rules, access control, URL rewriting, flexible origin, etc., to meet the acceleration needs of different business scenarios. Automating CDN rule engine configuration through Terraform can ensure standardized and consistent rule configuration, improving operational efficiency. This best practice will introduce how to use Terraform to automatically configure CDN rule engine rules.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [CDN Rule Engine Rule Resource (huaweicloud_cdn_rule_engine_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cdn_rule_engine_rule)

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create CDN Rule Engine Rule Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a CDN rule engine rule resource:

```hcl
variable "domain_name" {
  description = "The accelerated domain name to which the rule engine rule belongs"
  type        = string
}

variable "rule_name" {
  description = "The name of the rule engine rule"
  type        = string
}

variable "rule_status" {
  description = "Whether to enable the rule engine rule"
  type        = string
  default     = "on"

  validation {
    condition     = contains(["on", "off"], var.rule_status)
    error_message = "The rule_status must be one of: on, off."
  }
}

variable "rule_priority" {
  description = "The priority of the rule engine rule"
  type        = number
  default     = 1
}

variable "conditions" {
  description = "The trigger conditions of the rule engine rule, in JSON format"
  type        = string
  default     = ""
}

variable "cache_rule" {
  description = "The cache rule configuration"
  type = object({
    ttl           = number
    ttl_unit      = string
    follow_origin = optional(string)
    force_cache   = optional(string)
  })
  default = null
}

variable "access_control" {
  description = "The access control configuration"
  type = object({
    type = string
  })
  default = null
}

variable "http_response_headers" {
  description = "The list of HTTP response header configurations"
  type = list(object({
    name   = string
    value  = string
    action = string
  }))
  default = []
}

variable "browser_cache_rule" {
  description = "The browser cache rule configuration"
  type = object({
    cache_type = string
  })
  default = null
}

variable "request_url_rewrite" {
  description = "The access URL rewrite configuration"
  type = object({
    execution_mode = string
    redirect_url   = string
  })
  default = null
}

variable "flexible_origins" {
  description = "The list of flexible origin configurations"
  type = list(object({
    sources_type      = string
    ip_or_domain      = string
    priority          = number
    weight            = number
    http_port         = optional(number)
    https_port        = optional(number)
    origin_protocol   = optional(string)
    host_name         = optional(string)
    obs_bucket_type   = optional(string)
    bucket_access_key = optional(string)
    bucket_secret_key = optional(string)
    bucket_region     = optional(string)
    bucket_name       = optional(string)
  }))
  default = []
}

variable "origin_request_headers" {
  description = "The list of origin request header configurations"
  type = list(object({
    action = string
    name   = string
    value  = optional(string)
  }))
  default = []
}

variable "origin_request_url_rewrite" {
  description = "The origin request URL rewrite configuration"
  type = object({
    rewrite_type = string
    target_url   = string
  })
  default = null
}

variable "origin_range" {
  description = "The origin range configuration"
  type = object({
    status = string
  })
  default = null
}

variable "request_limit_rule" {
  description = "The request rate limit configuration"
  type = object({
    limit_rate_after = number
    limit_rate_value = number
  })
  default = null
}

variable "error_code_cache" {
  description = "The list of error code cache configurations"
  type = list(object({
    code = number
    ttl  = number
  }))
  default = []
}

# Create CDN rule engine rule resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_cdn_rule_engine_rule" "test" {
  domain_name = var.domain_name
  name        = var.rule_name
  status      = var.rule_status
  priority    = var.rule_priority
  conditions  = var.conditions != "" ? var.conditions : null

  dynamic "actions" {
    for_each = var.cache_rule != null ? [var.cache_rule] : []

    content {
      cache_rule {
        ttl           = actions.value.ttl
        ttl_unit      = actions.value.ttl_unit
        follow_origin = lookup(actions.value, "follow_origin", null)
        force_cache   = lookup(actions.value, "force_cache", null)
      }
    }
  }

  dynamic "actions" {
    for_each = var.access_control != null ? [var.access_control] : []

    content {
      access_control {
        type = actions.value.type
      }
    }
  }

  dynamic "actions" {
    for_each = length(var.http_response_headers) > 0 ? var.http_response_headers : []

    content {
      http_response_header {
        name   = actions.value.name
        value  = actions.value.value
        action = actions.value.action
      }
    }
  }

  dynamic "actions" {
    for_each = var.browser_cache_rule != null ? [var.browser_cache_rule] : []

    content {
      browser_cache_rule {
        cache_type = actions.value.cache_type
      }
    }
  }

  dynamic "actions" {
    for_each = var.request_url_rewrite != null ? [var.request_url_rewrite] : []

    content {
      request_url_rewrite {
        execution_mode = actions.value.execution_mode
        redirect_url   = actions.value.redirect_url
      }
    }
  }

  dynamic "actions" {
    for_each = length(var.flexible_origins) > 0 ? var.flexible_origins : []

    content {
      flexible_origin {
        sources_type      = actions.value.sources_type
        ip_or_domain      = actions.value.ip_or_domain
        priority          = actions.value.priority
        weight            = actions.value.weight
        http_port         = lookup(actions.value, "http_port", null)
        https_port        = lookup(actions.value, "https_port", null)
        origin_protocol   = lookup(actions.value, "origin_protocol", null)
        host_name         = lookup(actions.value, "host_name", null)
        obs_bucket_type   = lookup(actions.value, "obs_bucket_type", null)
        bucket_access_key = lookup(actions.value, "bucket_access_key", null)
        bucket_secret_key = lookup(actions.value, "bucket_secret_key", null)
        bucket_region     = lookup(actions.value, "bucket_region", null)
        bucket_name       = lookup(actions.value, "bucket_name", null)
      }
    }
  }

  dynamic "actions" {
    for_each = length(var.origin_request_headers) > 0 ? var.origin_request_headers : []

    content {
      origin_request_header {
        action = actions.value.action
        name   = actions.value.name
        value  = lookup(actions.value, "value", null)
      }
    }
  }

  dynamic "actions" {
    for_each = var.origin_request_url_rewrite != null ? [var.origin_request_url_rewrite] : []

    content {
      origin_request_url_rewrite {
        rewrite_type = actions.value.rewrite_type
        target_url   = actions.value.target_url
      }
    }
  }

  dynamic "actions" {
    for_each = var.origin_range != null ? [var.origin_range] : []

    content {
      origin_range {
        status = actions.value.status
      }
    }
  }

  dynamic "actions" {
    for_each = var.request_limit_rule != null ? [var.request_limit_rule] : []

    content {
      request_limit_rule {
        limit_rate_after = actions.value.limit_rate_after
        limit_rate_value = actions.value.limit_rate_value
      }
    }
  }

  dynamic "actions" {
    for_each = length(var.error_code_cache) > 0 ? var.error_code_cache : []

    content {
      error_code_cache {
        code = actions.value.code
        ttl  = actions.value.ttl
      }
    }
  }

  lifecycle {
    ignore_changes = [
      conditions,
    ]
  }
}
```

**Parameter Description**:
- **domain_name**: The accelerated domain name to which the rule belongs, assigned by referencing the input variable domain_name
- **name**: The rule name, assigned by referencing the input variable rule_name, length is 1-50 characters
- **status**: Whether to enable the rule, assigned by referencing the input variable rule_status, valid values: on (enabled), off (disabled), default value is "on"
- **priority**: The rule priority, assigned by referencing the input variable rule_priority, value range is 1-100, default value is 1
- **conditions**: The rule trigger conditions, assigned by referencing the input variable conditions, JSON format string, default value is empty string
- **actions.cache_rule**: Cache rule action, configured when cache_rule is not null
  - **ttl**: Cache time
  - **ttl_unit**: Cache time unit
  - **follow_origin**: Whether to follow origin server
  - **force_cache**: Whether to force cache
- **actions.access_control**: Access control action, configured when access_control is not null
  - **type**: Access control type
- **actions.http_response_header**: HTTP response header action, configured when http_response_headers list is not empty
  - **name**: Response header name
  - **value**: Response header value
  - **action**: Action type
- **actions.browser_cache_rule**: Browser cache rule action, configured when browser_cache_rule is not null
  - **cache_type**: Cache type
- **actions.request_url_rewrite**: Request URL rewrite action, configured when request_url_rewrite is not null
  - **execution_mode**: Execution mode
  - **redirect_url**: Redirect URL
- **actions.flexible_origin**: Flexible origin action, configured when flexible_origins list is not empty
  - **sources_type**: Origin server type
  - **ip_or_domain**: IP address or domain name
  - **priority**: Priority
  - **weight**: Weight
  - **http_port**: HTTP port
  - **https_port**: HTTPS port
  - **origin_protocol**: Origin protocol
  - **host_name**: Origin Host
  - **obs_bucket_type**: OBS bucket type
  - **bucket_access_key**: OBS bucket access key
  - **bucket_secret_key**: OBS bucket secret key
  - **bucket_region**: OBS bucket region
  - **bucket_name**: OBS bucket name
- **actions.origin_request_header**: Origin request header action, configured when origin_request_headers list is not empty
  - **action**: Action type
  - **name**: Request header name
  - **value**: Request header value
- **actions.origin_request_url_rewrite**: Origin request URL rewrite action, configured when origin_request_url_rewrite is not null
  - **rewrite_type**: Rewrite type
  - **target_url**: Target URL
- **actions.origin_range**: Origin Range action, configured when origin_range is not null
  - **status**: Status
- **actions.request_limit_rule**: Request rate limit action, configured when request_limit_rule is not null
  - **limit_rate_after**: Rate limit start value
  - **limit_rate_value**: Rate limit value
- **actions.error_code_cache**: Error code cache action, configured when error_code_cache list is not empty
  - **code**: Error code
  - **ttl**: Cache time

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Rule Engine Rule Configuration
domain_name   = "example.com"
rule_name     = "test-rule-engine"
rule_status   = "on"
rule_priority = 1

# Conditions Configuration (JSON format)
conditions = <<-JSON
{
  "match": {
    "logic": "and",
    "criteria": [
      {
        "match_target_type": "path",
        "match_type": "contains",
        "match_pattern": ["/api/"],
        "negate": false,
        "case_sensitive": true
      }
    ]
  }
}
JSON

# Origin Request URL Rewrite
origin_request_url_rewrite = {
  rewrite_type = "simple"
  target_url   = "/api/v2"
}

# Cache Rule Configuration
cache_rule = {
  ttl           = 10
  ttl_unit      = "m"
  follow_origin = "min_ttl"
  force_cache   = "off"
}

# Access Control Configuration
access_control = {
  type = "trust"
}

# Browser Cache Rule
browser_cache_rule = {
  cache_type = "follow_origin"
}

# Request URL Rewrite
request_url_rewrite = {
  execution_mode = "break"
  redirect_url   = "/new-path"
}

# Origin Range
origin_range = {
  status = "on"
}

# Request Limit Rule
request_limit_rule = {
  limit_rate_after = 2
  limit_rate_value = 1048576
}

# Origin Request Headers
origin_request_headers = [
  {
    action = "set"
    name   = "X-Real-IP"
    value  = "$realip_from_header"
  }
]

# Flexible Origins
flexible_origins = [
  {
    sources_type    = "domain"
    ip_or_domain    = "target.domain.com"
    priority        = 1
    weight          = 10
    http_port       = 80
    https_port      = 443
    origin_protocol = "follow"
    host_name       = "target.domain.com"
  }
]

# HTTP Response Headers
http_response_headers = [
  {
    name   = "Access-Control-Allow-Origin"
    value  = "*"
    action = "set"
  }
]

# Error Code Cache
error_code_cache = [
  {
    code = 400
    ttl  = 60
  }
]
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="domain_name=example.com" -var="rule_name=test-rule"`
2. Environment variables: `export TF_VAR_domain_name=example.com` and `export TF_VAR_rule_name=test-rule`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create a CDN rule engine rule:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the rule engine rule
4. Run `terraform show` to view the details of the created rule engine rule

> Note: Rule engine rules are processed in priority order. Each action type must be declared in a separate actions block. Conditions are specified in JSON format and must follow the API specification. Domain names cannot be updated after creation. Rule priority must be unique within the same domain. Flexible origin configurations support multiple origin types: ipaddr, domain, obs_bucket, third_bucket. Error code caching can help reduce origin server load for frequently occurring errors.

## Reference Information

- [Huawei Cloud CDN Product Documentation](https://support.huaweicloud.com/cdn/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Rule Engine](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cdn/rule-engine)
