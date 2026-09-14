# 部署公网域名

## 应用场景

云解析服务（Domain Name Service, DNS）是华为云提供的高可用、高性能的域名解析服务，支持公网域名解析和私网域名解析。通过创建公网域名，您可以将自有域名托管至华为云DNS，实现域名的智能解析、负载均衡与故障转移。

本最佳实践将介绍如何使用Terraform自动化部署DNS公网域名，包括域名创建、TTL配置、DNSSEC设置以及路由器关联。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [公网域名（huaweicloud_dns_zone）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dns_zone)

### 资源/数据源依赖关系

```
huaweicloud_dns_zone
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建公网域名

在TF文件（如main.tf）中添加以下脚本以创建公网域名：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建公网域名资源
variable "dns_public_zone_name" {
  description = "The name of the zone"
  type        = string
}

variable "dns_public_zone_email" {
  description = "The email address of the administrator managing the zone"
  type        = string
  default     = ""
}

variable "dns_public_zone_type" {
  description = "The type of zone"
  type        = string
  default     = "public"
}

variable "dns_public_zone_description" {
  description = "The description of the zone"
  type        = string
}

variable "dns_public_zone_ttl" {
  description = "The time to live (TTL) of the zone"
  type        = number
  default     = 300
}

variable "dns_public_zone_enterprise_project_id" {
  description = "The enterprise project ID of the zone"
  type        = string
  default     = ""
}

variable "dns_public_zone_status" {
  description = "The status of the zone"
  type        = string
  default     = "ENABLE"
}

variable "dns_public_zone_dnssec" {
  description = "Whether to enable DNSSEC for a public zone"
  type        = string
  default     = "DISABLE"
}

variable "dns_public_zone_router" {
  description = "The list of the router of the zone"
  type        = list(object({
    router_id     = string
    router_region = string
  }))
  default     = []
}

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
- **name**：通过引用输入变量 dns_public_zone_name 进行赋值，为域名的名称，注意名称末尾需要包含 `.`
- **email**：通过引用输入变量 dns_public_zone_email 进行赋值，为管理该域名的管理员邮箱地址
- **zone_type**：通过引用输入变量 dns_public_zone_type 进行赋值，为域名的类型，默认为 `public`
- **description**：通过引用输入变量 dns_public_zone_description 进行赋值，为域名的描述信息
- **ttl**：通过引用输入变量 dns_public_zone_ttl 进行赋值，为域名的缓存时间（TTL），默认为 300
- **enterprise_project_id**：通过引用输入变量 dns_public_zone_enterprise_project_id 进行赋值，为域名所属的企业项目ID
- **status**：通过引用输入变量 dns_public_zone_status 进行赋值，为域名的状态，默认为 `ENABLE`
- **dnssec**：通过引用输入变量 dns_public_zone_dnssec 进行赋值，用于设置是否开启公网域名的DNSSEC，默认为 `DISABLE`
- **router**：通过引用输入变量 dns_public_zone_router 进行赋值，为域名关联的路由器（VPC）列表，包含 `router_id`（关联VPC的ID）和 `router_region`（VPC所在的区域）

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 根据脚本变量填写；敏感信息使用占位符
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

1. 命令行参数：`terraform apply -var="dns_public_zone_name=my-zone.com"`
2. 环境变量：`export TF_VAR_dns_public_zone_name=my-zone.com`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建公网域名
4. 运行 `terraform show` 查看已创建的公网域名

## 参考信息

- [华为云DNS产品文档](https://support.huaweicloud.com/dns/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DNS公网域名最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dns/zone)
