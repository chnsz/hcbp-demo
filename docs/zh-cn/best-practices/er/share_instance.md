# 部署共享实例

## 应用场景

企业路由器（Enterprise Router, ER）是华为云提供的高性能、高可用的企业级路由器服务，支持多VPC互通、专线接入、VPN连接等企业级网络功能。ER服务提供灵活的路由策略和丰富的网络连接能力，满足企业复杂的网络架构需求。

ER共享实例是ER服务的重要功能，允许一个账户（所有者）将ER实例共享给其他账户（接受者），实现跨账户的网络资源共享。通过RAM（Resource Access Manager）服务，所有者可以精确控制共享权限，接受者可以安全地使用共享的ER实例。这种配置适用于多账户环境、合作伙伴网络共享、成本优化等场景。本最佳实践将介绍如何使用Terraform自动化部署ER共享实例，包括ER实例创建、RAM资源共享、跨账户VPC连接和附件接受。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [ER可用区列表查询数据源（data.huaweicloud_er_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/er_availability_zones)
- [RAM资源权限查询数据源（data.huaweicloud_ram_resource_permissions）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/ram_resource_permissions)
- [RAM资源共享邀请查询数据源（data.huaweicloud_ram_resource_share_invitations）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/ram_resource_share_invitations)

### 资源

- [ER实例资源（huaweicloud_er_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/er_instance)
- [RAM资源共享资源（huaweicloud_ram_resource_share）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ram_resource_share)
- [RAM资源共享接受者资源（huaweicloud_ram_resource_share_accepter）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ram_resource_share_accepter)
- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [ER VPC连接资源（huaweicloud_er_vpc_attachment）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/er_vpc_attachment)
- [ER连接接受者资源（huaweicloud_er_attachment_accepter）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/er_attachment_accepter)

### 资源/数据源依赖关系

```
data.huaweicloud_er_availability_zones.test
    └── huaweicloud_er_instance.test

huaweicloud_er_instance.test
    ├── data.huaweicloud_ram_resource_permissions.test
    │   └── huaweicloud_ram_resource_share.test
    └── huaweicloud_er_attachment_accepter.test

huaweicloud_ram_resource_share.test
    └── data.huaweicloud_ram_resource_share_invitations.test
        └── huaweicloud_ram_resource_share_accepter.test

huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test
        └── huaweicloud_er_vpc_attachment.test

huaweicloud_er_vpc_attachment.test
    └── huaweicloud_er_attachment_accepter.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 配置多账户Provider

在TF文件（如main.tf）中添加以下脚本以配置多账户Provider：

```hcl
# 所有者账户Provider配置
provider "huaweicloud" {
  alias      = "owner"
  region     = var.region_name
  access_key = var.access_key
  secret_key = var.secret_key
}

# 接受者账户Provider配置
provider "huaweicloud" {
  alias      = "principal"
  region     = var.region_name
  access_key = var.principal_access_key
  secret_key = var.principal_secret_key
}
```

**参数说明**：
- **owner**：所有者账户Provider，用于创建和共享ER实例
- **principal**：接受者账户Provider，用于接受共享的ER实例

### 3. 通过数据源查询ER可用区信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建ER实例：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的ER可用区信息，用于创建ER实例
data "huaweicloud_er_availability_zones" "test" {
  provider = huaweicloud.owner
}
```

**参数说明**：
- **provider**：指定使用所有者账户Provider进行查询

### 4. 创建ER实例

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建ER实例资源：

```hcl
variable "instance_name" {
  description = "ER实例名称"
  type        = string
}

variable "instance_asn" {
  description = "ER实例ASN号"
  type        = number
  default     = 64512
}

variable "instance_description" {
  description = "ER实例描述"
  type        = string
  default     = "The ER instance to share with other accounts"
}

variable "instance_enable_default_propagation" {
  description = "是否启用默认传播"
  type        = bool
  default     = true
}

variable "instance_enable_default_association" {
  description = "是否启用默认关联"
  type        = bool
  default     = true
}

variable "instance_auto_accept_shared_attachments" {
  description = "是否自动接受共享附件"
  type        = bool
  default     = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ER实例资源
resource "huaweicloud_er_instance" "test" {
  provider = huaweicloud.owner

  availability_zones = slice(data.huaweicloud_er_availability_zones.test.names, 0, 1)

  name        = var.instance_name
  asn         = var.instance_asn
  description = var.instance_description

  enable_default_propagation     = var.instance_enable_default_propagation
  enable_default_association     = var.instance_enable_default_association
  auto_accept_shared_attachments = var.instance_auto_accept_shared_attachments
}
```

**参数说明**：
- **provider**：指定使用所有者账户Provider创建资源
- **availability_zones**：可用区列表，使用ER可用区列表查询数据源的第一个结果
- **name**：实例名称，通过引用输入变量instance_name进行赋值
- **asn**：ASN号，通过引用输入变量instance_asn进行赋值
- **description**：实例描述，通过引用输入变量instance_description进行赋值
- **enable_default_propagation**：启用默认传播，通过引用输入变量instance_enable_default_propagation进行赋值
- **enable_default_association**：启用默认关联，通过引用输入变量instance_enable_default_association进行赋值
- **auto_accept_shared_attachments**：自动接受共享附件，通过引用输入变量instance_auto_accept_shared_attachments进行赋值

### 5. 查询RAM资源权限

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建RAM资源共享：

```hcl
# 获取ER实例的RAM资源权限信息，用于创建资源共享
data "huaweicloud_ram_resource_permissions" "test" {
  provider = huaweicloud.owner

  resource_type = "er:instances"

  depends_on = [huaweicloud_er_instance.test]
}
```

**参数说明**：
- **provider**：指定使用所有者账户Provider进行查询
- **resource_type**：资源类型，设置为"er:instances"表示ER实例资源
- **depends_on**：依赖关系，确保ER实例创建完成后再查询权限

### 6. 创建RAM资源共享

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建RAM资源共享资源：

```hcl
variable "resource_share_name" {
  description = "资源共享名称"
  type        = string
  default     = "resource-share-er"
}

variable "principal_account_id" {
  description = "接受者账户ID"
  type        = string
}

variable "owner_account_id" {
  description = "所有者账户ID"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RAM资源共享资源
resource "huaweicloud_ram_resource_share" "test" {
  provider = huaweicloud.owner

  name          = var.resource_share_name
  principals    = [var.principal_account_id]
  resource_urns = ["er:${var.region_name}:${var.owner_account_id}:instances:${huaweicloud_er_instance.test.id}"]

  permission_ids = data.huaweicloud_ram_resource_permissions.test.permissions[*].id
}
```

**参数说明**：
- **provider**：指定使用所有者账户Provider创建资源
- **name**：资源共享名称，通过引用输入变量resource_share_name进行赋值
- **principals**：接受者账户ID列表，通过引用输入变量principal_account_id进行赋值
- **resource_urns**：资源URN列表，包含ER实例的完整资源标识符
- **permission_ids**：权限ID列表，使用RAM资源权限查询数据源的权限ID

### 7. 查询RAM资源共享邀请

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于接受资源共享：

```hcl
# 获取待接受的RAM资源共享邀请信息，用于接受资源共享
data "huaweicloud_ram_resource_share_invitations" "test" {
  provider = huaweicloud.principal

  status = "pending"

  depends_on = [huaweicloud_ram_resource_share.test]
}
```

**参数说明**：
- **provider**：指定使用接受者账户Provider进行查询
- **status**：邀请状态，设置为"pending"表示待接受的邀请
- **depends_on**：依赖关系，确保资源共享创建完成后再查询邀请

### 8. 接受RAM资源共享

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建RAM资源共享接受者资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RAM资源共享接受者资源
resource "huaweicloud_ram_resource_share_accepter" "test" {
  provider = huaweicloud.principal

  resource_share_invitation_id = try([for v in data.huaweicloud_ram_resource_share_invitations.test.resource_share_invitations : v.id if v.resource_share_id == huaweicloud_ram_resource_share.test.id][0], "")
  action                       = "accept"

  # 接受邀请后，再次查询data.huaweicloud_ram_resource_share_invitations将为空
  # 此资源是一次性资源，添加ignore_changes防止执行terraform plan时资源变更
  lifecycle {
    ignore_changes = [
      resource_share_invitation_id,
    ]
  }
}
```

**参数说明**：
- **provider**：指定使用接受者账户Provider创建资源
- **resource_share_invitation_id**：资源共享邀请ID，通过查询结果匹配对应的邀请ID
- **action**：操作类型，设置为"accept"表示接受邀请
- **lifecycle.ignore_changes**：生命周期忽略变更，防止资源ID变更导致的问题

### 9. 创建接受者VPC

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建接受者VPC资源：

```hcl
variable "principal_vpc_name" {
  description = "接受者VPC名称"
  type        = string
}

variable "principal_vpc_cidr" {
  description = "接受者VPC的CIDR块"
  type        = string
  default     = "192.168.0.0/16"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建接受者VPC资源
resource "huaweicloud_vpc" "test" {
  provider = huaweicloud.principal

  name = var.principal_vpc_name
  cidr = var.principal_vpc_cidr
}
```

**参数说明**：
- **provider**：指定使用接受者账户Provider创建资源
- **name**：VPC名称，通过引用输入变量principal_vpc_name进行赋值
- **cidr**：VPC的CIDR块，通过引用输入变量principal_vpc_cidr进行赋值

### 10. 创建接受者VPC子网

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建接受者VPC子网资源：

```hcl
variable "principal_subnet_name" {
  description = "接受者VPC子网名称"
  type        = string
}

variable "principal_subnet_cidr" {
  description = "接受者VPC子网的CIDR块"
  type        = string
  default     = ""
  nullable    = false
}

variable "principal_subnet_gateway_ip" {
  description = "接受者VPC子网的网关IP"
  type        = string
  default     = ""
  nullable    = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建接受者VPC子网资源
resource "huaweicloud_vpc_subnet" "test" {
  provider = huaweicloud.principal

  vpc_id     = huaweicloud_vpc.test.id
  name       = var.principal_subnet_name
  cidr       = var.principal_subnet_cidr != "" ? var.principal_subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.principal_subnet_gateway_ip != "" ? var.principal_subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**参数说明**：
- **provider**：指定使用接受者账户Provider创建资源
- **vpc_id**：VPC ID，通过引用接受者VPC资源（huaweicloud_vpc.test）的ID进行赋值
- **name**：子网名称，通过引用输入变量principal_subnet_name进行赋值
- **cidr**：子网CIDR块，优先使用输入变量，如果为空则通过cidrsubnet函数计算
- **gateway_ip**：网关IP，优先使用输入变量，如果为空则通过cidrhost函数计算

### 11. 创建ER VPC连接

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建ER VPC连接资源：

```hcl
variable "attachment_name" {
  description = "ER VPC连接名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ER VPC连接资源
resource "huaweicloud_er_vpc_attachment" "test" {
  provider = huaweicloud.principal

  instance_id = huaweicloud_er_instance.test.id
  vpc_id      = huaweicloud_vpc.test.id
  subnet_id   = huaweicloud_vpc_subnet.test.id
  name        = var.attachment_name

  depends_on = [huaweicloud_ram_resource_share_accepter.test]
}
```

**参数说明**：
- **provider**：指定使用接受者账户Provider创建资源
- **instance_id**：ER实例ID，通过引用ER实例资源（huaweicloud_er_instance.test）的ID进行赋值
- **vpc_id**：VPC ID，通过引用接受者VPC资源（huaweicloud_vpc.test）的ID进行赋值
- **subnet_id**：子网ID，通过引用接受者VPC子网资源（huaweicloud_vpc_subnet.test）的ID进行赋值
- **name**：连接名称，通过引用输入变量attachment_name进行赋值
- **depends_on**：依赖关系，确保RAM资源共享接受完成后再创建VPC连接

### 12. 接受ER连接

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建ER连接接受者资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ER连接接受者资源
resource "huaweicloud_er_attachment_accepter" "test" {
  provider = huaweicloud.owner

  instance_id   = huaweicloud_er_instance.test.id
  attachment_id = huaweicloud_er_vpc_attachment.test.id
  action        = "accept"
}
```

**参数说明**：
- **provider**：指定使用所有者账户Provider创建资源
- **instance_id**：ER实例ID，通过引用ER实例资源（huaweicloud_er_instance.test）的ID进行赋值
- **attachment_id**：附件ID，通过引用ER VPC连接资源（huaweicloud_er_vpc_attachment.test）的ID进行赋值
- **action**：操作类型，设置为"accept"表示接受连接

### 13. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 接受者账户认证信息
principal_access_key = "your_principal_access_key"
principal_secret_key = "your_principal_secret_key"

# ER实例配置
instance_name = "tf_test_er_instance"

# RAM资源共享配置
resource_share_name  = "tf_test_resource_share"
principal_account_id = "your_principal_account_id"
owner_account_id     = "your_owner_account_id"

# 接受者VPC配置
principal_vpc_name    = "tf_test_er_instance_vpc"
principal_subnet_name = "tf_test_er_instance_subnet"

# ER VPC连接配置
attachment_name = "tf_test_er_attachment"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="instance_name=my-er" -var="principal_account_id=123456"`
2. 环境变量：`export TF_VAR_instance_name=my-er`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 14. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建ER共享实例
4. 运行 `terraform show` 查看已创建的ER共享实例

## 参考信息

- [华为云ER产品文档](https://support.huaweicloud.com/er/index.html)
- [华为云RAM产品文档](https://support.huaweicloud.com/ram/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [ER最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/er)
