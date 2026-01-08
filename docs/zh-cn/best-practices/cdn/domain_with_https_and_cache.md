# 部署HTTPS和缓存域名

## 应用场景

内容分发网络（Content Delivery Network，CDN）域名是CDN服务提供的加速域名配置功能，用于为网站、下载、视频等业务提供内容加速服务。通过配置CDN域名，您可以将源站内容分发到全球边缘节点，提高用户访问速度。通过配置HTTPS，您可以保障数据传输的安全性。通过配置缓存规则，您可以优化缓存策略，提高缓存命中率。通过Terraform自动化创建CDN域名，可以确保域名配置的规范化和一致性，提高运维效率。本最佳实践将介绍如何使用Terraform自动化创建CDN域名，包括HTTPS和缓存规则的配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [CDN域名资源（huaweicloud_cdn_domain）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cdn_domain)

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建CDN域名资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CDN域名资源：

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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CDN域名资源
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

**参数说明**：
- **name**：加速域名名称，通过引用输入变量domain_name进行赋值
- **type**：域名业务类型，通过引用输入变量domain_type进行赋值，可选值：web（网站加速）、download（文件下载加速）、video（点播加速）、wholeSite（全站加速），默认值为"web"
- **service_area**：服务范围，通过引用输入变量service_area进行赋值，可选值：mainland_china（中国大陆）、outside_mainland_china（中国大陆境外）、global（全球），默认值为"mainland_china"
- **sources.origin**：源站地址，通过引用输入变量origin_server进行赋值，可以是IP地址或域名
- **sources.origin_type**：源站类型，通过引用输入变量origin_type进行赋值，可选值：ipaddr（IP地址）、domain（域名）、obs_bucket（OBS桶域名），默认值为"ipaddr"
- **sources.active**：主备状态，设置为1表示主源站
- **sources.http_port**：源站HTTP端口，通过引用输入变量http_port进行赋值，默认值为80
- **sources.https_port**：源站HTTPS端口，通过引用输入变量https_port进行赋值，默认值为443
- **configs.origin_protocol**：回源协议，通过引用输入变量origin_protocol进行赋值，可选值：http、https、follow（跟随），默认值为"http"
- **configs.ipv6_enable**：是否开启IPv6，通过引用输入变量ipv6_enable进行赋值，默认值为false
- **configs.range_based_retrieval_enabled**：是否开启Range回源，通过引用输入变量range_based_retrieval_enabled进行赋值，默认值为false
- **configs.description**：域名描述，通过引用输入变量domain_description进行赋值，默认值为空字符串
- **configs.https_settings.certificate_name**：证书名称，当https_enabled为true时通过引用输入变量certificate_name进行赋值
- **configs.https_settings.certificate_source**：证书来源，当https_enabled为true时通过引用输入变量certificate_source进行赋值，可选值：0（华为云托管证书）、2（自有证书），默认值为"0"
- **configs.https_settings.certificate_body**：证书内容，当https_enabled为true且使用自有证书时，通过file函数读取certificate_body_path文件内容
- **configs.https_settings.private_key**：私钥内容，当https_enabled为true且使用自有证书时，通过file函数读取private_key_path文件内容
- **configs.https_settings.https_enabled**：是否开启HTTPS，通过引用输入变量https_enabled进行赋值，默认值为false
- **configs.https_settings.http2_enabled**：是否开启HTTP/2，通过引用输入变量http2_enabled进行赋值，仅在https_enabled为true时有效，默认值为false
- **configs.https_settings.ocsp_stapling_status**：OCSP装订状态，通过引用输入变量ocsp_stapling_status进行赋值，仅在https_enabled为true时有效，可选值：on（开启）、off（关闭），默认值为"off"
- **cache_settings.rules.rule_type**：缓存规则类型，可选值：all（所有文件）、file_extension（文件后缀）、catalog（目录）、full_path（全路径）、home_page（首页）
- **cache_settings.rules.content**：缓存规则匹配内容，根据rule_type设置不同的匹配内容
- **cache_settings.rules.ttl**：缓存时间，单位为ttl_type指定的单位，最大缓存时间为365天
- **cache_settings.rules.ttl_type**：缓存时间单位，可选值：s（秒）、m（分钟）、h（小时）、d（天）
- **cache_settings.rules.priority**：缓存规则优先级，数值越大优先级越高，取值范围1-100，权重值必须唯一
- **cache_settings.rules.url_parameter_type**：URL参数类型，可选值：del_params（忽略指定URL参数）、reserve_params（保留指定URL参数）、ignore_url_params（忽略所有URL参数）、full_url（保留所有URL参数），默认值为"full_url"
- **cache_settings.rules.url_parameter_value**：URL参数值，多个参数用逗号分隔，最多设置10个参数，当url_parameter_type为del_params或reserve_params时必填
- **tags**：域名标签，通过引用输入变量domain_tags进行赋值，默认值为空映射

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 域名配置
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

# HTTPS配置
https_enabled                 = true
certificate_name              = "terraform_test_cert"
certificate_source            = "0"
certificate_body_path         = "/path/to/your/certificate.crt"
private_key_path              = "/path/to/your/private.key"
http2_enabled                 = true
ocsp_stapling_status          = "on"

# 缓存规则配置
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

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="domain_name=example.com" -var="origin_server=192.168.1.100"`
2. 环境变量：`export TF_VAR_domain_name=example.com` 和 `export TF_VAR_origin_server=192.168.1.100`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建CDN域名：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建CDN域名
4. 运行 `terraform show` 查看已创建的CDN域名详情

> 注意：CDN域名创建可能需要几分钟时间完成。在更新域名配置之前，请确保状态值为**online**。服务范围不能在中国大陆和境外之间切换。SSL证书文件应妥善保管，不要提交到版本控制系统。缓存规则按优先级顺序处理（数值越小优先级越高）。域名名称在您的华为云账户内必须唯一。

## 参考信息

- [华为云CDN产品文档](https://support.huaweicloud.com/cdn/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [HTTPS和缓存域名最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cdn/domain-with-https-and-cache)
