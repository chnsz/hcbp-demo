# 部署主备Redis实例

## 应用场景

分布式缓存服务（Distributed Cache Service, DCS）是华为云提供的高性能、高可用的内存数据库服务，支持Redis、Memcached等主流缓存引擎。DCS服务提供多种实例规格和部署模式，包括单机、主备、集群等，满足不同规模和场景的缓存需求。

主备Redis实例是DCS服务中的高可用部署模式，通过主备架构实现数据冗余和故障自动切换，确保缓存服务的高可用性。主备实例支持自动故障检测和切换，当主节点发生故障时，备节点会自动接管服务，保证业务连续性。本最佳实践将介绍如何使用Terraform自动化部署DCS主备Redis实例，包括VPC创建、实例配置、备份策略和白名单管理。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DCS规格列表查询数据源（data.huaweicloud_dcs_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dcs_flavors)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [DCS实例资源（huaweicloud_dcs_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_instance)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones.test
    └── huaweicloud_dcs_instance.test

data.huaweicloud_dcs_flavors.test
    └── huaweicloud_dcs_instance.test

huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test
        └── huaweicloud_dcs_instance.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建VPC

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

### 3. 创建VPC子网

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC子网资源：

```hcl
variable "subnet_name" {
  description = "子网名称"
  type        = string
}

variable "subnet_cidr" {
  description = "子网的CIDR块"
  type        = string
  default     = ""
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "子网的网关IP地址"
  type        = string
  default     = ""
  nullable    = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**参数说明**：
- **vpc_id**：VPC ID，通过引用VPC资源（huaweicloud_vpc.test）的ID进行赋值
- **name**：子网名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网CIDR块，优先使用输入变量，如果为空则通过cidrsubnet函数计算
- **gateway_ip**：网关IP，优先使用输入变量，如果为空则通过cidrhost函数计算

### 4. 通过数据源查询可用区信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建DCS实例：

```hcl
variable "availability_zones" {
  description = "Redis实例所属的可用区列表"
  type        = list(string)
  default     = []
  nullable    = false
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建DCS实例
data "huaweicloud_availability_zones" "test" {
  count = length(var.availability_zones) < 1 ? 1 : 0
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行可用区列表查询数据源，仅当 `var.availability_zones` 长度小于1时创建数据源

### 5. 通过数据源查询DCS规格信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建DCS实例：

```hcl
variable "instance_flavor_id" {
  description = "Redis实例规格ID"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_cache_mode" {
  description = "Redis实例缓存模式"
  type        = string
  default     = "ha"
}

variable "instance_capacity" {
  description = "Redis实例容量（GB）"
  type        = number
  default     = 4
}

variable "instance_engine_version" {
  description = "Redis实例引擎版本"
  type        = string
  default     = "5.0"
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下符合条件的所有DCS规格信息，用于创建DCS实例
data "huaweicloud_dcs_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  cache_mode     = var.instance_cache_mode
  capacity       = var.instance_capacity
  engine_version = var.instance_engine_version
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行DCS规格列表查询数据源，仅当 `var.instance_flavor_id` 为空时创建数据源
- **cache_mode**：缓存模式，通过引用输入变量instance_cache_mode进行赋值，设置为"ha"表示主备模式
- **capacity**：容量，通过引用输入变量instance_capacity进行赋值
- **engine_version**：引擎版本，通过引用输入变量instance_engine_version进行赋值

### 6. 创建DCS主备Redis实例

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建DCS实例资源：

```hcl
variable "instance_name" {
  description = "Redis实例名称"
  type        = string
}

variable "enterprise_project_id" {
  description = "Redis实例所属企业项目ID"
  type        = string
  default     = null
}

variable "instance_password" {
  description = "Redis实例密码"
  type        = string
  sensitive   = true
  default     = null
}

variable "instance_backup_policy" {
  description = "Redis实例备份策略"
  type = object({
    backup_type = optional(string, "auto")
    backup_at   = list(number)
    begin_at    = string
    save_days   = optional(number, null)
    period_type = optional(string, null)
  })
  default = null
}

variable "instance_whitelists" {
  description = "Redis实例白名单"
  type = list(object({
    group_name = string
    ip_address = list(string)
  }))
  default  = []
  nullable = false
}

variable "instance_parameters" {
  description = "Redis实例参数"
  type = list(object({
    id    = string
    name  = string
    value = string
  }))
  default  = []
  nullable = false
}

variable "instance_tags" {
  description = "Redis实例标签"
  type        = map(string)
  default     = {}
}

variable "instance_rename_commands" {
  description = "Redis实例重命名命令"
  type        = map(string)
  default     = {}
}

variable "charging_mode" {
  description = "Redis实例计费模式"
  type        = string
  default     = "postPaid"
}

variable "period_unit" {
  description = "计费周期单位"
  type        = string
  default     = null
}

variable "period" {
  description = "Redis实例计费周期"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "是否启用自动续费"
  type        = string
  default     = "false"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DCS实例资源
resource "huaweicloud_dcs_instance" "test" {
  name                  = var.instance_name
  engine                = "Redis"
  enterprise_project_id = var.enterprise_project_id
  engine_version        = var.instance_engine_version
  capacity              = var.instance_capacity
  flavor                = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_dcs_flavors.test[0].flavors[0].name, null)
  availability_zones    = length(var.availability_zones) > 0 ? var.availability_zones : try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 2), null)
  vpc_id                = huaweicloud_vpc.test.id
  subnet_id             = huaweicloud_vpc_subnet.test.id
  password              = var.instance_password

  dynamic "backup_policy" {
    for_each = var.instance_backup_policy != null ? [var.instance_backup_policy] : []
    content {
      backup_type = lookup(backup_policy.value, "backup_type", null)
      save_days   = lookup(backup_policy.value, "save_days", null)
      period_type = lookup(backup_policy.value, "period_type", null)
      backup_at   = backup_policy.value["backup_at"]
      begin_at    = backup_policy.value["begin_at"]
    }
  }

  dynamic "whitelists" {
    for_each = var.instance_whitelists
    content {
      group_name = whitelists.value["group_name"]
      ip_address = whitelists.value["ip_address"]
    }
  }

  dynamic "parameters" {
    for_each = var.instance_parameters
    content {
      id    = parameters.value["id"]
      name  = parameters.value["name"]
      value = parameters.value["value"]
    }
  }

  tags            = var.instance_tags
  rename_commands = var.instance_rename_commands

  charging_mode = var.charging_mode
  period_unit   = var.period_unit
  period        = var.period
  auto_renew    = var.auto_renew

  # If you want to change the `flavor` or `availability_zones`, you need to delete it from the `lifecycle.ignore_changes`.
  lifecycle {
    ignore_changes = [
      flavor,
      availability_zones,
    ]
  }
}
```

**参数说明**：
- **name**：实例名称，通过引用输入变量instance_name进行赋值
- **engine**：引擎类型，设置为"Redis"
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值
- **engine_version**：引擎版本，通过引用输入变量instance_engine_version进行赋值
- **capacity**：容量，通过引用输入变量instance_capacity进行赋值
- **flavor**：规格，优先使用输入变量，如果为空则使用DCS规格列表查询数据源的第一个结果
- **availability_zones**：可用区列表，优先使用输入变量，如果为空则使用可用区列表查询数据源的前两个结果
- **vpc_id**：VPC ID，通过引用VPC资源（huaweicloud_vpc.test）的ID进行赋值
- **subnet_id**：子网ID，通过引用VPC子网资源（huaweicloud_vpc_subnet.test）的ID进行赋值
- **password**：密码，通过引用输入变量instance_password进行赋值
- **backup_policy.backup_type**：备份类型，通过引用备份策略中的backup_type字段进行赋值
- **backup_policy.save_days**：保存天数，通过引用备份策略中的save_days字段进行赋值
- **backup_policy.period_type**：周期类型，通过引用备份策略中的period_type字段进行赋值
- **backup_policy.backup_at**：备份时间，通过引用备份策略中的backup_at字段进行赋值
- **backup_policy.begin_at**：开始时间，通过引用备份策略中的begin_at字段进行赋值
- **whitelists.group_name**：白名单组名，通过引用白名单列表中的group_name字段进行赋值
- **whitelists.ip_address**：IP地址列表，通过引用白名单列表中的ip_address字段进行赋值
- **parameters.id**：参数ID，通过引用参数列表中的id字段进行赋值
- **parameters.name**：参数名称，通过引用参数列表中的name字段进行赋值
- **parameters.value**：参数值，通过引用参数列表中的value字段进行赋值
- **tags**：标签，通过引用输入变量instance_tags进行赋值
- **rename_commands**：重命名命令，通过引用输入变量instance_rename_commands进行赋值
- **charging_mode**：计费模式，通过引用输入变量charging_mode进行赋值
- **period_unit**：计费周期单位，通过引用输入变量period_unit进行赋值
- **period**：计费周期，通过引用输入变量period进行赋值
- **auto_renew**：自动续费，通过引用输入变量auto_renew进行赋值
- **lifecycle.ignore_changes**：生命周期忽略变更，忽略flavor和availability_zones的变更

### 7. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 网络配置
vpc_name    = "tf_test_dcs_instance_vpc"
subnet_name = "tf_test_dcs_instance_subnet"

# DCS实例配置
instance_name     = "tf_test_dcs_instance"
instance_password = "YourRedisInstancePassword!"

# 备份策略配置
instance_backup_policy = {
  backup_type = "auto"
  backup_at   = [1, 3, 4, 5, 6]
  begin_at    = "02:00-04:00"
  save_days   = 7
}

# 白名单配置
instance_whitelists = [
  {
    group_name = "test-group1"
    ip_address = ["192.168.10.100", "192.168.0.0/24"]
  },
  {
    group_name = "test-group2"
    ip_address = ["172.16.10.100", "172.16.0.0/24"]
  }
]

# 参数配置
instance_parameters = [
  {
    id    = "1"
    name  = "timeout"
    value = "500"
  },
  {
    id    = "3"
    name  = "hash-max-ziplist-entries"
    value = "4096"
  }
]

# 标签配置
instance_tags = {
  foo = "bar"
}
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="instance_name=my-redis"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 8. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DCS主备Redis实例
4. 运行 `terraform show` 查看已创建的DCS主备Redis实例

## 参考信息

- [华为云DCS产品文档](https://support.huaweicloud.com/dcs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DCS主备Redis实例最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-high-availability-instance)
