# 部署ACL规则配置

## 应用场景

云防火墙（Cloud Firewall，CFW）是新一代的云原生防火墙，提供云上互联网边界和VPC边界的防护，包括实时入侵检测与防御、全局统一访问控制、全流量分析可视化、日志审计与溯源分析等能力。CFW支持基于五元组、域名、应用及IP地址组、服务组设置ACL访问控制策略，实现对南北向和东西向流量的精细化管控。

本最佳实践将介绍如何使用Terraform自动化部署CFW ACL规则配置，包括查询已有防火墙信息，创建IP地址组、服务组、域名组，以及基于IP地址、域名和地址组/服务组的多类型ACL访问控制规则。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [CFW防火墙列表查询数据源（data.huaweicloud_cfw_firewalls）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/cfw_firewalls)

### 资源

- [CFW IP地址组资源（huaweicloud_cfw_address_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cfw_address_group)
- [CFW服务组资源（huaweicloud_cfw_service_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cfw_service_group)
- [CFW域名组资源（huaweicloud_cfw_domain_name_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cfw_domain_name_group)
- [CFW ACL规则资源（huaweicloud_cfw_acl_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cfw_acl_rule)

### 资源/数据源依赖关系

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

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询防火墙信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建CFW地址组、服务组、域名组及ACL规则：

```hcl
variable "fw_instance_id" {
  description = "The firewall instance ID"
  type        = string
  default     = ""
  nullable    = false
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下防火墙信息，用于创建CFW地址组、服务组、域名组及ACL规则
data "huaweicloud_cfw_firewalls" "test" {
  fw_instance_id = var.fw_instance_id != "" ? var.fw_instance_id : null
}

locals {
  fw_instance_id = var.fw_instance_id != "" ? var.fw_instance_id : try(data.huaweicloud_cfw_firewalls.test.records[0].fw_instance_id, null)
  object_id      = try(data.huaweicloud_cfw_firewalls.test.records[0].protect_objects[0].object_id, null)
}
```

**参数说明**：
- **fw_instance_id**：防火墙实例ID，通过引用输入变量fw_instance_id进行赋值，当值为空字符串时设置为null以查询默认防火墙

### 3. 创建IP地址组

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CFW IP地址组资源：

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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CFW IP地址组资源
resource "huaweicloud_cfw_address_group" "test" {
  object_id   = local.object_id
  name        = var.address_group_name
  description = var.address_group_description
}
```

**参数说明**：
- **object_id**：防护对象ID，根据防火墙列表查询数据源（data.huaweicloud_cfw_firewalls）返回结果中的protect_objects进行赋值
- **name**：IP地址组名称，通过引用输入变量address_group_name进行赋值
- **description**：IP地址组描述，通过引用输入变量address_group_description进行赋值，默认为空字符串

### 4. 创建服务组

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CFW服务组资源：

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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CFW服务组资源
resource "huaweicloud_cfw_service_group" "test" {
  object_id   = local.object_id
  name        = var.service_group_name
  description = var.service_group_description
}
```

**参数说明**：
- **object_id**：防护对象ID，根据防火墙列表查询数据源（data.huaweicloud_cfw_firewalls）返回结果中的protect_objects进行赋值
- **name**：服务组名称，通过引用输入变量service_group_name进行赋值
- **description**：服务组描述，通过引用输入变量service_group_description进行赋值，默认为空字符串

### 5. 创建域名组

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CFW域名组资源：

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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CFW域名组资源
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

**参数说明**：
- **fw_instance_id**：防火墙实例ID，根据防火墙列表查询数据源（data.huaweicloud_cfw_firewalls）返回结果进行赋值
- **object_id**：防护对象ID，根据防火墙列表查询数据源（data.huaweicloud_cfw_firewalls）返回结果中的protect_objects进行赋值
- **name**：域名组名称，通过引用输入变量domain_name_group_name进行赋值
- **type**：域名组类型，通过引用输入变量domain_name_group_type进行赋值，默认为0
- **description**：域名组描述，通过引用输入变量domain_name_group_description进行赋值，默认为空字符串
- **domain_names**：域名列表，通过动态块 `dynamic "domain_names"` 根据输入变量domain_name_group_domains创建
  - **domain_name**：域名，通过引用输入变量中的domain_name进行赋值
  - **description**：域名描述，通过引用输入变量中的description进行赋值

### 6. 创建基于IP地址的ACL规则

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建基于IP地址的CFW ACL规则资源：

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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建基于IP地址的CFW ACL规则资源
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

**参数说明**：
- **name**：ACL规则名称，通过引用输入变量acl_rule_ip_name进行赋值
- **object_id**：防护对象ID，根据防火墙列表查询数据源（data.huaweicloud_cfw_firewalls）返回结果中的protect_objects进行赋值
- **description**：ACL规则描述，通过引用输入变量acl_rule_ip_description进行赋值，默认为空字符串
- **type**：ACL规则类型，通过引用输入变量acl_rule_type进行赋值，0表示互联网规则，1表示VPC规则，2表示NAT规则，默认为0
- **address_type**：ACL规则地址类型，通过引用输入变量acl_rule_address_type进行赋值，0表示IPv4，1表示IPv6，默认为0
- **action_type**：ACL规则动作类型，通过引用输入变量acl_rule_action_type进行赋值，0表示允许，1表示拒绝，默认为0
- **long_connect_enable**：是否启用长连接，通过引用输入变量acl_rule_long_connect_enable进行赋值，0表示禁用，1表示启用，默认为0
- **status**：ACL规则状态，通过引用输入变量acl_rule_status进行赋值，0表示禁用，1表示启用，默认为1
- **applications**：ACL规则应用列表，通过引用输入变量acl_rule_applications进行赋值，默认为["HTTPS"]
- **source_addresses**：源IP地址列表，通过引用输入变量acl_rule_source_addresses进行赋值，默认为["1.1.1.1"]
- **destination_addresses**：目的IP地址列表，通过引用输入变量acl_rule_destination_addresses进行赋值，默认为["1.1.1.2"]
- **custom_services**：自定义服务配置块
  - **protocol**：自定义服务协议类型，通过引用输入变量acl_rule_custom_service_protocol进行赋值，6表示TCP，17表示UDP，默认为6
  - **source_port**：自定义服务源端口，通过引用输入变量acl_rule_custom_service_source_port进行赋值，默认为"81"
  - **dest_port**：自定义服务目的端口，通过引用输入变量acl_rule_custom_service_dest_port进行赋值，默认为"82"
- **sequence**：规则优先级配置块
  - **top**：规则置顶优先级，设置为1
- **tags**：关联到资源的标签键值对，通过引用输入变量tags进行赋值

### 7. 创建基于域名的ACL规则

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建基于域名的CFW ACL规则资源：

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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建基于域名的CFW ACL规则资源
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

**参数说明**：
- **name**：ACL规则名称，通过引用输入变量acl_rule_domain_name进行赋值
- **object_id**：防护对象ID，根据防火墙列表查询数据源（data.huaweicloud_cfw_firewalls）返回结果中的protect_objects进行赋值
- **description**：ACL规则描述，通过引用输入变量acl_rule_domain_description进行赋值，默认为空字符串
- **type**：ACL规则类型，通过引用输入变量acl_rule_type进行赋值
- **address_type**：ACL规则地址类型，通过引用输入变量acl_rule_address_type进行赋值
- **action_type**：ACL规则动作类型，通过引用输入变量acl_rule_action_type进行赋值
- **long_connect_enable**：是否启用长连接，通过引用输入变量acl_rule_long_connect_enable进行赋值
- **status**：ACL规则状态，通过引用输入变量acl_rule_status进行赋值
- **direction**：ACL规则方向，通过引用输入变量acl_rule_domain_direction进行赋值，0表示入方向，1表示出方向，默认为1
- **source_addresses**：源IP地址列表，通过引用输入变量acl_rule_source_addresses进行赋值
- **destination_domain_address_name**：目的域名地址名称，通过引用输入变量acl_rule_destination_domain_address_name进行赋值，默认为"*.baidu.com"
- **custom_services**：自定义服务配置块
  - **protocol**：自定义服务协议类型，通过引用输入变量acl_rule_custom_service_protocol进行赋值
  - **source_port**：自定义服务源端口，通过引用输入变量acl_rule_custom_service_source_port进行赋值
  - **dest_port**：自定义服务目的端口，通过引用输入变量acl_rule_custom_service_dest_port进行赋值
- **sequence**：规则优先级配置块
  - **top**：规则置顶优先级，设置为0
  - **dest_rule_id**：目标规则ID，引用前面创建的基于IP地址的ACL规则资源（huaweicloud_cfw_acl_rule.ip_based）的ID
- **tags**：关联到资源的标签键值对，通过引用输入变量tags进行赋值

### 8. 创建基于地址组和服务组的ACL规则

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建基于地址组和服务组的CFW ACL规则资源：

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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建基于地址组和服务组的CFW ACL规则资源
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

**参数说明**：
- **name**：ACL规则名称，通过引用输入变量acl_rule_group_name进行赋值
- **object_id**：防护对象ID，根据防火墙列表查询数据源（data.huaweicloud_cfw_firewalls）返回结果中的protect_objects进行赋值
- **description**：ACL规则描述，通过引用输入变量acl_rule_group_description进行赋值，默认为空字符串
- **type**：ACL规则类型，通过引用输入变量acl_rule_type进行赋值
- **address_type**：ACL规则地址类型，通过引用输入变量acl_rule_address_type进行赋值
- **action_type**：ACL规则动作类型，通过引用输入变量acl_rule_action_type进行赋值
- **long_connect_enable**：是否启用长连接，通过引用输入变量acl_rule_long_connect_enable进行赋值
- **status**：ACL规则状态，通过引用输入变量acl_rule_status进行赋值
- **source_address_groups**：源IP地址组ID列表，引用前面创建的CFW IP地址组资源（huaweicloud_cfw_address_group.test）的ID
- **destination_address_groups**：目的IP地址组ID列表，引用前面创建的CFW IP地址组资源（huaweicloud_cfw_address_group.test）的ID
- **custom_service_groups**：自定义服务组配置块
  - **protocols**：服务组协议类型列表，通过引用输入变量acl_rule_service_group_protocol进行赋值，默认为6（TCP）
  - **group_ids**：服务组ID列表，引用前面创建的CFW服务组资源（huaweicloud_cfw_service_group.test）的ID
- **sequence**：规则优先级配置块
  - **bottom**：规则置底优先级，设置为1
- **tags**：关联到资源的标签键值对，通过引用输入变量tags进行赋值

### 9. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 防火墙实例
fw_instance_id = "your_firewall_instance_id"

# IP地址组配置
address_group_name = "tf_test_address_group"

# 服务组配置
service_group_name = "tf_test_service_group"

# 域名组配置
domain_name_group_name = "tf_test_domain_name_group"

# ACL规则配置
acl_rule_ip_name     = "tf_test_acl_rule_ip"
acl_rule_domain_name = "tf_test_acl_rule_domain"
acl_rule_group_name  = "tf_test_acl_rule_group"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="fw_instance_id=your-firewall-id" -var="acl_rule_ip_name=tf_test_acl_rule_ip"`
2. 环境变量：`export TF_VAR_acl_rule_ip_name=tf_test_acl_rule_ip`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 10. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建CFW ACL规则配置
4. 运行 `terraform show` 查看已创建的CFW ACL规则配置

## 参考信息

- [华为云CFW产品文档](https://support.huaweicloud.com/cfw/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CFW ACL规则配置最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cfw/acl-rule-config)
