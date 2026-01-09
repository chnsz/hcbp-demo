# 部署规则引擎

## 应用场景

内容分发网络（Content Delivery Network，CDN）规则引擎是CDN服务提供的灵活规则配置功能，用于根据不同的请求条件执行相应的动作，实现精细化的CDN加速控制。通过配置规则引擎，您可以根据路径、参数、请求头等条件匹配请求，并执行缓存规则、访问控制、URL重写、灵活回源等多种动作，满足不同业务场景的加速需求。通过Terraform自动化配置CDN规则引擎，可以确保规则配置的规范化和一致性，提高运维效率。本最佳实践将介绍如何使用Terraform自动化配置CDN规则引擎规则。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [CDN规则引擎规则资源（huaweicloud_cdn_rule_engine_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cdn_rule_engine_rule)

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建CDN规则引擎规则资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CDN规则引擎规则资源：

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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CDN规则引擎规则资源
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

**参数说明**：
- **domain_name**：规则所属的加速域名，通过引用输入变量domain_name进行赋值
- **name**：规则名称，通过引用输入变量rule_name进行赋值，长度为1-50个字符
- **status**：是否启用规则，通过引用输入变量rule_status进行赋值，可选值：on（启用）、off（禁用），默认值为"on"
- **priority**：规则优先级，通过引用输入变量rule_priority进行赋值，取值范围1-100，默认值为1
- **conditions**：规则触发条件，通过引用输入变量conditions进行赋值，JSON格式字符串，默认值为空字符串
- **actions.cache_rule**：缓存规则动作，当cache_rule不为null时配置
  - **ttl**：缓存时间
  - **ttl_unit**：缓存时间单位
  - **follow_origin**：是否遵循源站
  - **force_cache**：是否强制缓存
- **actions.access_control**：访问控制动作，当access_control不为null时配置
  - **type**：访问控制类型
- **actions.http_response_header**：HTTP响应头动作，当http_response_headers列表不为空时配置
  - **name**：响应头名称
  - **value**：响应头值
  - **action**：动作类型
- **actions.browser_cache_rule**：浏览器缓存规则动作，当browser_cache_rule不为null时配置
  - **cache_type**：缓存类型
- **actions.request_url_rewrite**：请求URL重写动作，当request_url_rewrite不为null时配置
  - **execution_mode**：执行模式
  - **redirect_url**：重定向URL
- **actions.flexible_origin**：灵活回源动作，当flexible_origins列表不为空时配置
  - **sources_type**：源站类型
  - **ip_or_domain**：IP地址或域名
  - **priority**：优先级
  - **weight**：权重
  - **http_port**：HTTP端口
  - **https_port**：HTTPS端口
  - **origin_protocol**：回源协议
  - **host_name**：回源Host
  - **obs_bucket_type**：OBS桶类型
  - **bucket_access_key**：OBS桶访问密钥
  - **bucket_secret_key**：OBS桶密钥
  - **bucket_region**：OBS桶区域
  - **bucket_name**：OBS桶名称
- **actions.origin_request_header**：回源请求头动作，当origin_request_headers列表不为空时配置
  - **action**：动作类型
  - **name**：请求头名称
  - **value**：请求头值
- **actions.origin_request_url_rewrite**：回源请求URL重写动作，当origin_request_url_rewrite不为null时配置
  - **rewrite_type**：重写类型
  - **target_url**：目标URL
- **actions.origin_range**：回源Range动作，当origin_range不为null时配置
  - **status**：状态
- **actions.request_limit_rule**：请求限速动作，当request_limit_rule不为null时配置
  - **limit_rate_after**：限速起始值
  - **limit_rate_value**：限速值
- **actions.error_code_cache**：错误码缓存动作，当error_code_cache列表不为空时配置
  - **code**：错误码
  - **ttl**：缓存时间

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 规则引擎规则配置
domain_name   = "example.com"
rule_name     = "test-rule-engine"
rule_status   = "on"
rule_priority = 1

# 条件配置（JSON格式）
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

# 回源请求URL重写
origin_request_url_rewrite = {
  rewrite_type = "simple"
  target_url   = "/api/v2"
}

# 缓存规则配置
cache_rule = {
  ttl           = 10
  ttl_unit      = "m"
  follow_origin = "min_ttl"
  force_cache   = "off"
}

# 访问控制配置
access_control = {
  type = "trust"
}

# 浏览器缓存规则
browser_cache_rule = {
  cache_type = "follow_origin"
}

# 请求URL重写
request_url_rewrite = {
  execution_mode = "break"
  redirect_url   = "/new-path"
}

# 回源Range
origin_range = {
  status = "on"
}

# 请求限速规则
request_limit_rule = {
  limit_rate_after = 2
  limit_rate_value = 1048576
}

# 回源请求头
origin_request_headers = [
  {
    action = "set"
    name   = "X-Real-IP"
    value  = "$realip_from_header"
  }
]

# 灵活回源
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

# HTTP响应头
http_response_headers = [
  {
    name   = "Access-Control-Allow-Origin"
    value  = "*"
    action = "set"
  }
]

# 错误码缓存
error_code_cache = [
  {
    code = 400
    ttl  = 60
  }
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="domain_name=example.com" -var="rule_name=test-rule"`
2. 环境变量：`export TF_VAR_domain_name=example.com` 和 `export TF_VAR_rule_name=test-rule`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建CDN规则引擎规则：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建规则引擎规则
4. 运行 `terraform show` 查看已创建的规则引擎规则详情

> 注意：规则引擎规则按优先级顺序处理。每种动作类型必须在单独的actions块中声明。条件以JSON格式指定，必须遵循API规范。域名名称在创建后无法更新。规则优先级在同一域名内必须唯一。灵活回源配置支持多种源站类型：ipaddr、domain、obs_bucket、third_bucket。错误码缓存可以帮助减少频繁出现的错误对源站服务器的负载。

## 参考信息

- [华为云CDN产品文档](https://support.huaweicloud.com/cdn/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [规则引擎最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cdn/rule-engine)
