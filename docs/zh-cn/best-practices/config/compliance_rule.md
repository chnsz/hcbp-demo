# 部署合规规则

## 应用场景

配置审计（Config）是华为云提供的一站式合规管理服务，帮助用户持续监控和评估云资源的配置合规性。Config服务提供预置的合规规则包和自定义规则，支持多种合规框架和标准，帮助企业建立完善的合规管理体系。

合规规则是Config服务的核心功能，用于定义和执行具体的合规检查策略。通过合规规则，企业可以监控云资源的配置是否符合安全、合规要求，及时发现和修复配置风险。合规规则支持多种资源类型和检查条件，包括标签检查、配置验证、安全策略等，为企业提供全面的合规管理解决方案。本最佳实践将介绍如何使用Terraform自动化部署Config合规规则，包括VPC创建、ECS实例创建、合规规则配置和规则评估。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [ECS实例资源（huaweicloud_compute_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [RMS策略分配资源（huaweicloud_rms_policy_assignment）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rms_policy_assignment)
- [RMS策略分配评估资源（huaweicloud_rms_policy_assignment_evaluate）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rms_policy_assignment_evaluate)

### 资源/数据源依赖关系

```
huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test
        └── huaweicloud_compute_instance.test
            └── huaweicloud_rms_policy_assignment.test
                └── huaweicloud_rms_policy_assignment_evaluate.test
    └── huaweicloud_networking_secgroup.test
        └── huaweicloud_compute_instance.test
            └── huaweicloud_rms_policy_assignment.test
                └── huaweicloud_rms_policy_assignment_evaluate.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建VPC资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC资源：

```hcl
variable "vpc_name" {
  description = "VPC名称"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC的CIDR块"
  type        = string
  default     = "192.168.0.0/16"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量vpc_name进行赋值
- **cidr**：VPC的CIDR块，通过引用输入变量vpc_cidr进行赋值

### 3. 创建VPC子网资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC子网资源：

```hcl
variable "subnet_name" {
  description = "子网名称"
  type        = string
}

variable "subnet_cidr" {
  description = "子网的CIDR块，如果未指定，则在现有CIDR地址块中计算一个子网CIDR"
  type        = string
  default     = ""
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "子网的网关IP，如果未指定，则在现有CIDR地址块中计算一个网关IP"
  type        = string
  default     = ""
  nullable    = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**参数说明**：
- **vpc_id**：VPC ID，通过引用VPC资源（huaweicloud_vpc.test）的ID进行赋值
- **name**：子网名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网的CIDR块，通过引用输入变量subnet_cidr进行赋值，如果为空则自动计算
- **gateway_ip**：子网的网关IP，通过引用输入变量subnet_gateway_ip进行赋值，如果为空则自动计算

### 4. 创建安全组资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建安全组资源：

```hcl
variable "security_group_name" {
  description = "安全组名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组名称，通过引用输入变量security_group_name进行赋值
- **delete_default_rules**：是否删除默认规则，设置为true

### 5. 创建ECS实例资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建ECS实例资源：

```hcl
variable "ecs_instance_name" {
  description = "ECS实例名称"
  type        = string
}

variable "ecs_image_name" {
  description = "用于创建ECS实例的镜像名称"
  type        = string
  default     = "Ubuntu 20.04 server 64bit"
}

variable "ecs_flavor_name" {
  description = "ECS实例规格名称"
  type        = string
  default     = "s6.small.1"
}

variable "availability_zone" {
  description = "ECS实例所在的可用区"
  type        = string
}

variable "ecs_tags" {
  description = "ECS实例标签"
  type        = map(string)
  default = {
    "Owner" = "terraform"
    "Env"   = "test"
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ECS实例资源
resource "huaweicloud_compute_instance" "test" {
  name               = var.ecs_instance_name
  image_name         = var.ecs_image_name
  flavor_name        = var.ecs_flavor_name
  security_group_ids = [huaweicloud_networking_secgroup.test.id]
  availability_zone  = var.availability_zone

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }

  tags = var.ecs_tags
}
```

**参数说明**：
- **name**：实例名称，通过引用输入变量ecs_instance_name进行赋值
- **image_name**：镜像名称，通过引用输入变量ecs_image_name进行赋值
- **flavor_name**：规格名称，通过引用输入变量ecs_flavor_name进行赋值
- **security_group_ids**：安全组ID列表，通过引用安全组资源（huaweicloud_networking_secgroup.test）的ID进行赋值
- **availability_zone**：可用区，通过引用输入变量availability_zone进行赋值
- **network**：网络配置，通过引用VPC子网资源（huaweicloud_vpc_subnet.test）的ID进行赋值
- **tags**：标签，通过引用输入变量ecs_tags进行赋值

### 6. 创建合规规则

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建合规规则资源：

```hcl
variable "policy_assignment_name" {
  description = "策略分配名称"
  type        = string
}

variable "policy_assignment_description" {
  description = "策略分配描述"
  type        = string
  default     = "Check if ECS instances have required tags"
}

variable "policy_definition_id" {
  description = "策略定义ID"
  type        = string
}

variable "policy_assignment_policy_filter" {
  description = "用于过滤资源的配置"

  type = list(object({
    region            = string
    resource_provider = string
    resource_type     = string
    resource_id       = string
    tag_key           = string
    tag_value         = string
  }))

  default = []
}

variable "policy_assignment_parameters" {
  description = "策略分配参数"
  type        = map(string)
  default = {
    "tagKeys" = "[\"Owner\",\"Env\"]"
  }
}

variable "policy_assignment_tags" {
  description = "策略分配标签"
  type        = map(string)
  default = {
    "Owner" = "terraform"
    "Env"   = "test"
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建合规规则资源
resource "huaweicloud_rms_policy_assignment" "test" {
  name                 = var.policy_assignment_name
  description          = var.policy_assignment_description
  policy_definition_id = var.policy_definition_id

  dynamic "policy_filter" {
    for_each = var.policy_assignment_policy_filter

    content {
      region            = policy_filter.value.region
      resource_provider = policy_filter.value.resource_provider
      resource_type     = policy_filter.value.resource_type
      resource_id       = policy_filter.value.resource_id
      tag_key           = policy_filter.value.tag_key
      tag_value         = policy_filter.value.tag_value
    }
  }

  parameters = var.policy_assignment_parameters

  tags = var.policy_assignment_tags
}
```

**参数说明**：
- **name**：策略分配名称，通过引用输入变量policy_assignment_name进行赋值
- **description**：策略分配描述，通过引用输入变量policy_assignment_description进行赋值
- **policy_definition_id**：策略定义ID，通过引用输入变量policy_definition_id进行赋值
- **policy_filter**：策略过滤器，动态创建资源过滤条件
  - **region**：区域，通过引用过滤器配置中的region进行赋值
  - **resource_provider**：资源提供者，通过引用过滤器配置中的resource_provider进行赋值
  - **resource_type**：资源类型，通过引用过滤器配置中的resource_type进行赋值
  - **resource_id**：资源ID，通过引用过滤器配置中的resource_id进行赋值
  - **tag_key**：标签键，通过引用过滤器配置中的tag_key进行赋值
  - **tag_value**：标签值，通过引用过滤器配置中的tag_value进行赋值
- **parameters**：参数，通过引用输入变量policy_assignment_parameters进行赋值
- **tags**：标签，通过引用输入变量policy_assignment_tags进行赋值

### 7. 评估合规规则

在TF文件（如main.tf）中添加以下脚本以告知Terraform评估合规规则：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下评估合规规则
resource "huaweicloud_rms_policy_assignment_evaluate" "test" {
  policy_assignment_id = huaweicloud_rms_policy_assignment.test.id
}
```

**参数说明**：
- **policy_assignment_id**：策略分配ID，通过引用合规规则资源（huaweicloud_rms_policy_assignment.test）的ID进行赋值

### 8. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 网络配置
vpc_name               = "tf_test_vpc"
vpc_cidr               = "192.168.0.0/16"
subnet_name            = "tf_test_subnet"
subnet_cidr            = "192.168.0.0/24"
subnet_gateway_ip      = "192.168.0.1"
security_group_name    = "tf_test_security_group"

# ECS实例配置
ecs_instance_name      = "tf_test_ecs"
ecs_image_name         = "Ubuntu 20.04 server 64bit"
ecs_flavor_name        = "s6.small.1"
availability_zone      = "cn-north-4a"
ecs_tags               = {
  "Owner" = "terraform"
  "Env"   = "test"
}

# 合规规则配置
policy_assignment_name = "tf_test_policy_assignment_name"
policy_assignment_description = "Check if ECS instances have required tags"
policy_definition_id   = "tf_test_policy_definition_id"
policy_assignment_policy_filter = [
  {
    region            = "cn-north-4"
    resource_provider = "hws.resource.type.ecs"
    resource_type     = "hws.resource.type.ecs"
    resource_id       = ""
    tag_key           = "Owner"
    tag_value         = "terraform"
  }
]
policy_assignment_parameters = {
  "tagKeys" = "[\"Owner\",\"Env\"]"
}
policy_assignment_tags = {
  "Owner" = "terraform"
  "Env"   = "test"
}
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="policy_assignment_name=my-policy"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 9. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建合规规则
4. 运行 `terraform show` 查看已创建的合规规则

## 参考信息

- [华为云Config产品文档](https://support.huaweicloud.com/rms/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Config合规规则最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/rms/policy-assignment)
