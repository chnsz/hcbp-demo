# 部署实例

## 应用场景

专属主机（Dedicated Host，DEH）是华为云提供的物理服务器资源，用于满足对资源独享、安全合规等有特殊要求的业务场景。通过创建专属主机实例，您可以获得物理服务器的完全控制权，实现资源的物理隔离，满足合规性要求。通过Terraform自动化创建专属主机实例，可以确保资源部署的规范化和一致性，提高运维效率。本最佳实践将介绍如何使用Terraform自动化创建专属主机实例。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 数据源

- [可用区数据源（huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [专属主机类型数据源（huaweicloud_deh_types）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/deh_types)

### 资源

- [专属主机实例资源（huaweicloud_deh_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/deh_instance)

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询数据源

在TF文件（如main.tf）中添加以下脚本以查询可用区和专属主机类型信息：

```hcl
# 查询可用区信息
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}

# 查询专属主机类型信息
data "huaweicloud_deh_types" "test" {
  count = var.deh_instance_host_type == "" ? 1 : 0

  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
}
```

**参数说明**：
- **availability_zone**：可用区名称，当availability_zone变量为空时查询
- **host_type**：专属主机类型，当deh_instance_host_type变量为空时查询

### 3. 创建专属主机实例资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建专属主机实例资源：

```hcl
variable "instance_name" {
  description = "The name of the dedicated host instance"
  type        = string
}

variable "availability_zone" {
  description = "The availability zone where the resources will be created"
  type        = string
  default     = ""
}

variable "deh_instance_host_type" {
  description = "The host type of the dedicated host"
  type        = string
  default     = ""
}

variable "deh_instance_auto_placement" {
  description = "Whether to enable auto placement for the dedicated host"
  type        = string
  default     = "on"
}

variable "enterprise_project_id" {
  description = "The enterprise project ID of the dedicated host"
  type        = string
  default     = null
}

variable "deh_instance_charging_mode" {
  description = "The charging mode of the dedicated host"
  type        = string
  default     = "prePaid"
}

variable "deh_instance_period_unit" {
  description = "The unit of the billing period of the dedicated host"
  type        = string
  default     = "month"
}

variable "deh_instance_period" {
  description = "The billing period of the dedicated host"
  type        = string
  default     = "1"
}

variable "deh_instance_auto_renew" {
  description = "Whether to enable auto renew for the dedicated host"
  type        = string
  default     = "false"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建专属主机实例资源
resource "huaweicloud_deh_instance" "test" {
  name                  = var.instance_name
  availability_zone     = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  host_type             = var.deh_instance_host_type != "" ? var.deh_instance_host_type : try(data.huaweicloud_deh_types.test[0].dedicated_host_types[0].host_type, null)
  auto_placement        = var.deh_instance_auto_placement
  enterprise_project_id = var.enterprise_project_id
  charging_mode         = var.deh_instance_charging_mode
  period_unit           = var.deh_instance_period_unit
  period                = var.deh_instance_period
  auto_renew            = var.deh_instance_auto_renew
}
```

**参数说明**：
- **name**：专属主机实例名称，通过引用输入变量instance_name进行赋值
- **availability_zone**：可用区名称，通过引用输入变量availability_zone或可用区数据源进行赋值
- **host_type**：专属主机类型，通过引用输入变量deh_instance_host_type或专属主机类型数据源进行赋值
- **auto_placement**：是否启用自动放置，通过引用输入变量deh_instance_auto_placement进行赋值，默认值为"on"
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，可选参数，默认值为null
- **charging_mode**：计费模式，通过引用输入变量deh_instance_charging_mode进行赋值，默认值为"prePaid"（包年包月）
- **period_unit**：计费周期单位，通过引用输入变量deh_instance_period_unit进行赋值，默认值为"month"（月）
- **period**：计费周期，通过引用输入变量deh_instance_period进行赋值，默认值为"1"
- **auto_renew**：是否自动续费，通过引用输入变量deh_instance_auto_renew进行赋值，默认值为"false"

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 专属主机实例配置
instance_name = "tf_test_deh_instance"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="instance_name=my_deh_instance"`
2. 环境变量：`export TF_VAR_instance_name=my_deh_instance`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建专属主机实例：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建专属主机实例
4. 运行 `terraform show` 查看已创建的专属主机实例详情

> 注意：专属主机实例创建后，可以在该专属主机上部署ECS实例。专属主机提供物理服务器的完全控制权，实现资源的物理隔离，满足合规性要求。通过设置auto_placement可以启用自动放置功能，系统会自动选择合适的物理服务器。

## 参考信息

- [华为云DEH产品文档](https://support.huaweicloud.com/deh/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [实例最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/deh/instance)
