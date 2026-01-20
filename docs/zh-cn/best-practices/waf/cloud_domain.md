# 部署云模式域名

## 应用场景

华为云Web应用防火墙（Web Application Firewall，WAF）云模式是一种基于共享资源的Web安全防护服务，可以为指定的域名提供Web攻击防护。通过配置WAF云模式域名，可以为您的网站提供针对SQL注入、XSS跨站脚本、网页木马上传、命令注入、恶意爬虫、CC攻击等多种常见Web攻击的防护能力。WAF云模式域名支持灵活的源站配置、SSL证书管理、自定义错误页面、超时设置和流量标记等功能，满足不同业务场景的安全防护需求。本最佳实践将介绍如何使用Terraform自动化部署一个WAF云模式域名，包括创建WAF云实例和域名配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [WAF云实例资源（huaweicloud_waf_cloud_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/waf_cloud_instance)
- [WAF域名资源（huaweicloud_waf_domain）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/waf_domain)

### 资源/数据源依赖关系

```text
huaweicloud_waf_cloud_instance
    └── huaweicloud_waf_domain
```

> 注意：WAF域名依赖于WAF云实例。在创建WAF域名之前，需要先创建WAF云实例。WAF云实例提供基础的安全防护能力，WAF域名则是在云实例基础上配置的具体防护域名。

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建WAF云实例

在TF文件（如main.tf）中添加以下脚本以创建WAF云实例：

```hcl
resource "huaweicloud_waf_cloud_instance" "test" {
  resource_spec_code = var.cloud_instance_resource_spec_code

  dynamic "bandwidth_expack_product" {
    for_each = var.cloud_instance_bandwidth_expack_product

    content {
      resource_size = bandwidth_expack_product.value["resource_size"]
    }
  }

  dynamic "domain_expack_product" {
    for_each = var.cloud_instance_domain_expack_product

    content {
      resource_size = domain_expack_product.value["resource_size"]
    }
  }

  dynamic "rule_expack_product" {
    for_each = var.cloud_instance_rule_expack_product

    content {
      resource_size = rule_expack_product.value["resource_size"]
    }
  }

  charging_mode         = var.cloud_instance_charging_mode
  period_unit           = var.cloud_instance_period_unit
  period                = var.cloud_instance_period
  auto_renew            = var.cloud_instance_auto_renew
  enterprise_project_id = var.enterprise_project_id
}
```

**参数说明**：
- **resource_spec_code**：资源规格代码，通过引用输入变量 `cloud_instance_resource_spec_code` 进行赋值，如"detection"（检测模式）或"premium"（专业版）
- **bandwidth_expack_product**：带宽扩展包配置，通过动态块 `dynamic "bandwidth_expack_product"` 根据输入变量 `cloud_instance_bandwidth_expack_product` 创建带宽扩展包
  - **resource_size**：资源大小，通过引用输入变量中的 `resource_size` 进行赋值
- **domain_expack_product**：域名扩展包配置，通过动态块 `dynamic "domain_expack_product"` 根据输入变量 `cloud_instance_domain_expack_product` 创建域名扩展包
  - **resource_size**：资源大小，通过引用输入变量中的 `resource_size` 进行赋值
- **rule_expack_product**：规则扩展包配置，通过动态块 `dynamic "rule_expack_product"` 根据输入变量 `cloud_instance_rule_expack_product` 创建规则扩展包
  - **resource_size**：资源大小，通过引用输入变量中的 `resource_size` 进行赋值
- **charging_mode**：计费模式，通过引用输入变量 `cloud_instance_charging_mode` 进行赋值，如"prePaid"（包年包月）或"postPaid"（按需计费）
- **period_unit**：订购周期单位，通过引用输入变量 `cloud_instance_period_unit` 进行赋值，如"month"（月）或"year"（年）
- **period**：订购周期，通过引用输入变量 `cloud_instance_period` 进行赋值
- **auto_renew**：是否自动续费，通过引用输入变量 `cloud_instance_auto_renew` 进行赋值，如"true"或"false"
- **enterprise_project_id**：企业项目ID，通过引用输入变量 `enterprise_project_id` 进行赋值，默认为"0"表示默认企业项目

### 3. 创建WAF域名

在TF文件（如main.tf）中添加以下脚本以创建WAF域名：

```hcl
resource "huaweicloud_waf_domain" "test" {
  domain                = var.cloud_domain
  certificate_id        = var.cloud_certificate_id
  certificate_name      = var.cloud_certificate_name
  proxy                 = var.cloud_proxy
  enterprise_project_id = var.enterprise_project_id
  description           = var.cloud_description
  website_name          = var.cloud_website_name
  protect_status        = var.cloud_protect_status
  forward_header_map    = var.cloud_forward_header_map

  dynamic "custom_page" {
    for_each = var.cloud_custom_page

    content {
      http_return_code = custom_page.value["http_return_code"]
      block_page_type  = custom_page.value["block_page_type"]
      page_content     = custom_page.value["page_content"]
    }
  }

  dynamic "timeout_settings" {
    for_each = var.cloud_timeout_settings

    content {
      connection_timeout = timeout_settings.value["connection_timeout"]
      read_timeout       = timeout_settings.value["read_timeout"]
      write_timeout      = timeout_settings.value["write_timeout"]
    }
  }

  dynamic "traffic_mark" {
    for_each = var.cloud_traffic_mark

    content {
      ip_tags     = traffic_mark.value["ip_tags"]
      session_tag = traffic_mark.value["session_tag"]
      user_tag    = traffic_mark.value["user_tag"]
    }
  }

  dynamic "server" {
    for_each = var.cloud_server

    content {
      client_protocol = server.value["client_protocol"]
      server_protocol = server.value["server_protocol"]
      address         = server.value["address"]
      port            = server.value["port"]
      type            = server.value["type"]
      weight          = server.value["weight"]
    }
  }

  depends_on = [
    huaweicloud_waf_cloud_instance.test
  ]
}
```

**参数说明**：
- **domain**：需要防护的域名，通过引用输入变量 `cloud_domain` 进行赋值
- **certificate_id**：SSL证书ID，通过引用输入变量 `cloud_certificate_id` 进行赋值，用于HTTPS访问
- **certificate_name**：SSL证书名称，通过引用输入变量 `cloud_certificate_name` 进行赋值
- **proxy**：是否开启代理，通过引用输入变量 `cloud_proxy` 进行赋值，true表示开启代理，false表示关闭代理
- **enterprise_project_id**：企业项目ID，通过引用输入变量 `enterprise_project_id` 进行赋值，默认为"0"表示默认企业项目
- **description**：域名描述，通过引用输入变量 `cloud_description` 进行赋值
- **website_name**：网站名称，通过引用输入变量 `cloud_website_name` 进行赋值
- **protect_status**：防护状态，通过引用输入变量 `cloud_protect_status` 进行赋值，0表示关闭防护，1表示开启防护
- **forward_header_map**：字段转发配置，通过引用输入变量 `cloud_forward_header_map` 进行赋值，用于自定义请求头转发
- **custom_page**：自定义错误页面配置，通过动态块 `dynamic "custom_page"` 根据输入变量 `cloud_custom_page` 创建自定义错误页面
  - **http_return_code**：HTTP返回码，通过引用输入变量中的 `http_return_code` 进行赋值
  - **block_page_type**：拦截页面类型，通过引用输入变量中的 `block_page_type` 进行赋值
  - **page_content**：页面内容，通过引用输入变量中的 `page_content` 进行赋值
- **timeout_settings**：超时设置配置，通过动态块 `dynamic "timeout_settings"` 根据输入变量 `cloud_timeout_settings` 创建超时设置
  - **connection_timeout**：连接超时时间，通过引用输入变量中的 `connection_timeout` 进行赋值
  - **read_timeout**：读取超时时间，通过引用输入变量中的 `read_timeout` 进行赋值
  - **write_timeout**：写入超时时间，通过引用输入变量中的 `write_timeout` 进行赋值
- **traffic_mark**：流量标记配置，通过动态块 `dynamic "traffic_mark"` 根据输入变量 `cloud_traffic_mark` 创建流量标记
  - **ip_tags**：IP标签列表，通过引用输入变量中的 `ip_tags` 进行赋值
  - **session_tag**：会话标签，通过引用输入变量中的 `session_tag` 进行赋值
  - **user_tag**：用户标签，通过引用输入变量中的 `user_tag` 进行赋值
- **server**：源站服务器配置列表，通过动态块 `dynamic "server"` 根据输入变量 `cloud_server` 创建源站服务器配置
  - **client_protocol**：客户端协议，通过引用输入变量中的 `client_protocol` 进行赋值，如"HTTP"或"HTTPS"
  - **server_protocol**：服务端协议，通过引用输入变量中的 `server_protocol` 进行赋值，如"HTTP"或"HTTPS"
  - **address**：源站服务器地址，通过引用输入变量中的 `address` 进行赋值，可以是IP地址或域名
  - **port**：源站服务器端口，通过引用输入变量中的 `port` 进行赋值
  - **type**：源站服务器类型，通过引用输入变量中的 `type` 进行赋值，如"ipv4"或"ipv6"
  - **weight**：源站服务器权重，通过引用输入变量中的 `weight` 进行赋值，用于负载均衡
- **depends_on**：显式依赖关系，确保WAF云实例在WAF域名之前创建

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# WAF云实例配置（必填）
cloud_instance_resource_spec_code = "detection"
cloud_instance_charging_mode      = "prePaid"
cloud_instance_period_unit        = "month"
cloud_instance_period             = 1

# WAF域名配置（必填）
cloud_domain = "demo-example-test.huawei.com"

# 源站服务器配置（必填）
cloud_server = [
  {
    client_protocol = "HTTP"
    server_protocol = "HTTP"
    address         = "119.8.0.17"
    port            = 8080
    type            = "ipv4"
    weight          = 1
  }
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="cloud_domain=demo-example-test.huawei.com"`
2. 环境变量：`export TF_VAR_cloud_domain=demo-example-test.huawei.com`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建WAF云模式域名及相关资源
4. 运行 `terraform show` 查看已创建的WAF云模式域名

## 参考信息

- [华为云WAF产品文档](https://support.huaweicloud.com/waf/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [云模式域名最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/waf/cloud-domain)
