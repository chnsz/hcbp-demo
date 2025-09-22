# 部署公网域名

## 应用场景

云解析服务（Domain Name Service, DNS）是华为云提供的高可用、高性能的域名解析服务，支持公网域名解析和私网域名解析。DNS服务提供智能解析、负载均衡、健康检查等功能，帮助用户实现域名的智能调度和故障转移。

公网域名是DNS服务中的核心功能，用于管理面向互联网的域名解析。通过公网域名，企业可以管理自己的网站域名、API域名、邮件服务器域名等，实现域名到IP地址的映射。公网域名支持多种记录类型，包括A记录、AAAA记录、CNAME记录、MX记录等，满足不同应用场景的解析需求。本最佳实践将介绍如何使用Terraform自动化部署DNS公网域名，包括域名创建、TTL配置、DNSSEC设置和路由器关联。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [DNS公网域名资源（huaweicloud_dns_zone）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dns_zone)

### 资源/数据源依赖关系

```
无依赖关系
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建DNS公网域名

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建DNS公网域名资源：

```hcl
variable "dns_public_zone_name" {
  description = "域名名称"
  type        = string
}

variable "dns_public_zone_email" {
  description = "管理域名的管理员邮箱地址"
  type        = string
  default     = ""
}

variable "dns_public_zone_type" {
  description = "域名类型"
  type        = string
  default     = "public"
}

variable "dns_public_zone_description" {
  description = "域名描述"
  type        = string
}

variable "dns_public_zone_ttl" {
  description = "域名的生存时间（TTL）"
  type        = number
  default     = 300
}

variable "dns_public_zone_enterprise_project_id" {
  description = "域名所属企业项目ID"
  type        = string
  default     = ""
}

variable "dns_public_zone_status" {
  description = "域名状态"
  type        = string
  default     = "ENABLE"
}

variable "dns_public_zone_dnssec" {
  description = "是否为公网域名启用DNSSEC"
  type        = string
  default     = "DISABLE"
}

variable "dns_public_zone_router" {
  description = "域名关联的路由器列表"
  type = list(object({
    router_id     = string
    router_region = string
  }))
  default = []
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DNS公网域名资源
resource "huaweicloud_dns_zone" "test" {
  name                  = var.dns_public_zone_name
  email                 = var.dns_public_zone_email
  zone_type             = var.dns_public_zone_type
  description           = var.dns_public_zone_description
  ttl                   = var.dns_public_zone_ttl
  enterprise_project_id = var.dns_public_zone_enterprise_project_id
  status                = var.dns_public_zone_status
  dnssec                = var.dns_public_zone_dnssec

  dynamic "router" {
    for_each = var.dns_public_zone_router
    content {
      router_id     = router.value.router_id
      router_region = router.value.router_region
    }
  }
}
```

**参数说明**：
- **name**：域名名称，通过引用输入变量dns_public_zone_name进行赋值
- **email**：管理域名的管理员邮箱地址，通过引用输入变量dns_public_zone_email进行赋值
- **zone_type**：域名类型，通过引用输入变量dns_public_zone_type进行赋值，默认为"public"（公网）
- **description**：域名描述，通过引用输入变量dns_public_zone_description进行赋值
- **ttl**：域名的生存时间（TTL），通过引用输入变量dns_public_zone_ttl进行赋值，默认为300秒
- **enterprise_project_id**：域名所属企业项目ID，通过引用输入变量dns_public_zone_enterprise_project_id进行赋值
- **status**：域名状态，通过引用输入变量dns_public_zone_status进行赋值，默认为"ENABLE"（启用）
- **dnssec**：是否为公网域名启用DNSSEC，通过引用输入变量dns_public_zone_dnssec进行赋值，默认为"DISABLE"（禁用）
- **router.router_id**：路由器ID，通过引用路由器列表中的router_id字段进行赋值
- **router.router_region**：路由器区域，通过引用路由器列表中的router_region字段进行赋值

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# DNS公网域名配置
dns_public_zone_name        = "tftest.yourname.com"
dns_public_zone_description = "tf_test_zone_desc"
dns_public_zone_ttl         = 3000
dns_public_zone_dnssec      = "ENABLE"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="dns_public_zone_name=example.com" -var="dns_public_zone_description=My Domain"`
2. 环境变量：`export TF_VAR_dns_public_zone_name=example.com`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DNS公网域名
4. 运行 `terraform show` 查看已创建的DNS公网域名

## 参考信息

- [华为云DNS产品文档](https://support.huaweicloud.com/dns/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DNS最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dns)
