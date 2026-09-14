# 部署Redis高可用实例

## 应用场景

分布式缓存服务（DCS）是华为云提供的高性能、高可用的内存数据库服务，支持Redis、Memcached等主流缓存引擎。Redis高可用实例通过主备、集群、Proxy集群或读写分离等部署模式，在单节点故障时自动切换，保障缓存服务的连续性与数据可靠性，适用于会话存储、热点数据缓存、消息队列等对可用性要求较高的业务场景。

本最佳实践将介绍如何使用Terraform自动化部署一个DCS Redis高可用实例，包括VPC与子网的创建、可用分区与产品规格的自动查询、实例的创建以及备份策略、白名单、参数、标签等配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用分区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DCS产品规格列表（data.huaweicloud_dcs_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dcs_flavors)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [DCS实例（huaweicloud_dcs_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_instance)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_dcs_instance

data.huaweicloud_dcs_flavors
    └── huaweicloud_dcs_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dcs_instance
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建VPC

在TF文件（如main.tf）中添加以下脚本以创建虚拟私有云：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云资源
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量 vpc_name 进行赋值
- **cidr**：VPC的网段，通过引用输入变量 vpc_cidr 进行赋值

### 3. 创建VPC子网

在TF文件（如main.tf）中添加以下脚本以创建虚拟私有云子网：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云子网资源
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "The gateway IP address of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**参数说明**：
- **vpc_id**：子网所属的VPC ID，引用前一步创建的VPC资源的ID
- **name**：子网名称，通过引用输入变量 subnet_name 进行赋值
- **cidr**：子网的网段，通过引用输入变量 subnet_cidr 进行赋值；若未指定则基于VPC网段自动划分子网
- **gateway_ip**：子网的网关IP，通过引用输入变量 subnet_gateway_ip 进行赋值；若未指定则基于子网网段自动计算

### 4. 查询可用分区列表

在TF文件（如main.tf）中添加以下脚本以查询可用分区列表：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用分区列表数据源
variable "availability_zones" {
  description = "The availability zones to which the Redis instance belongs"
  type        = list(string)
  default     = []
  nullable    = false
}

data "huaweicloud_availability_zones" "test" {
  count = length(var.availability_zones) < 1 ? 1 : 0
}
```

**参数说明**：
- **count**：当输入变量 availability_zones 未指定可用分区时，才执行该数据源查询

### 5. 查询DCS产品规格列表

在TF文件（如main.tf）中添加以下脚本以查询DCS产品规格列表：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询DCS产品规格列表数据源
variable "instance_flavor_id" {
  description = "The flavor ID of the Redis instance"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_cache_mode" {
  description = "The cache mode of the Redis instance"
  type        = string
  default     = "ha"
}

variable "instance_capacity" {
  description = "The capacity of the Redis instance (in GB)"
  type        = number
  default     = 4
}

variable "instance_engine_version" {
  description = "The engine version of the Redis instance"
  type        = string
  default     = "5.0"
}

data "huaweicloud_dcs_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  cache_mode     = var.instance_cache_mode
  capacity       = var.instance_capacity
  engine_version = var.instance_engine_version
}
```

**参数说明**：
- **count**：当输入变量 instance_flavor_id 未指定规格时，才执行该数据源查询
- **cache_mode**：缓存模式，通过引用输入变量 instance_cache_mode 进行赋值
- **capacity**：实例容量（GB），通过引用输入变量 instance_capacity 进行赋值
- **engine_version**：引擎版本，通过引用输入变量 instance_engine_version 进行赋值

### 6. 创建DCS Redis实例

在TF文件（如main.tf）中添加以下脚本以创建DCS Redis实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DCS实例资源
variable "instance_name" {
  description = "The name of the Redis instance"
  type        = string
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project to which the Redis instance belongs"
  type        = string
  default     = null
}

variable "instance_password" {
  description = "The password for the Redis instance"
  type        = string
  sensitive   = true
  default     = null
}

variable "instance_backup_policy" {
  description = "The backup policy of the Redis instance"
  type        = object({
    backup_type = optional(string, "auto")
    backup_at   = list(number)
    begin_at    = string
    save_days   = optional(number, null)
    period_type = optional(string, null)
  })
  default     = null
}

variable "instance_whitelists" {
  description = "The whitelists of the Redis instance"
  type        = list(object({
    group_name = string
    ip_address = list(string)
  }))
  default     = []
  nullable    = false
}

variable "instance_parameters" {
  description = "The parameters of the Redis instance"
  type        = list(object({
    id    = string
    name  = string
    value = string
  }))
  default     = []
  nullable    = false
}

variable "instance_tags" {
  description = "The tags of the Redis instance"
  type        = map(string)
  default     = {}
}

variable "instance_rename_commands" {
  description = "The rename commands of the Redis instance"
  type        = map(string)
  default     = {}
}

variable "charging_mode" {
  description = "The charging mode of the Redis instance"
  type        = string
  default     = "postPaid"
}

variable "period_unit" {
  description = "The unit of the period"
  type        = string
  default     = null
}

variable "period" {
  description = "The period of the Redis instance"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "Whether auto renew is enabled"
  type        = string
  default     = "false"
}

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
- **name**：实例名称，通过引用输入变量 instance_name 进行赋值
- **engine**：缓存引擎，固定为Redis
- **enterprise_project_id**：企业项目ID，通过引用输入变量 enterprise_project_id 进行赋值
- **engine_version**：引擎版本，通过引用输入变量 instance_engine_version 进行赋值
- **capacity**：实例容量（GB），通过引用输入变量 instance_capacity 进行赋值
- **flavor**：实例规格，优先使用输入变量 instance_flavor_id，未指定时引用DCS产品规格数据源查询结果
- **availability_zones**：可用分区列表，优先使用输入变量 availability_zones，未指定时引用可用分区数据源查询结果的前两个分区
- **vpc_id**：实例所属的VPC ID，引用前一步创建的VPC资源的ID
- **subnet_id**：实例所属的子网ID，引用前一步创建的子网资源的ID
- **password**：实例密码，通过引用输入变量 instance_password 进行赋值
- **backup_policy**：备份策略，通过引用输入变量 instance_backup_policy 进行赋值
- **whitelists**：白名单，通过引用输入变量 instance_whitelists 进行赋值
- **parameters**：实例参数，通过引用输入变量 instance_parameters 进行赋值
- **tags**：实例标签，通过引用输入变量 instance_tags 进行赋值
- **rename_commands**：重命名命令，通过引用输入变量 instance_rename_commands 进行赋值
- **charging_mode**：计费模式，通过引用输入变量 charging_mode 进行赋值
- **period_unit**：计费周期单位，通过引用输入变量 period_unit 进行赋值
- **period**：计费周期，通过引用输入变量 period 进行赋值
- **auto_renew**：是否自动续订，通过引用输入变量 auto_renew 进行赋值

### 7. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 资源变量
vpc_name               = "tf_test_dcs_instance_vpc"
subnet_name            = "tf_test_dcs_instance_subnet"
instance_name          = "tf_test_dcs_instance"
instance_password      = "YourRedisInstancePassword!"
instance_backup_policy = {
  backup_type = "auto"
  backup_at   = [1, 3, 4, 5, 6]
  begin_at    = "02:00-04:00"
  save_days   = 7
}

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

instance_tags = {
  foo = "bar"
}
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 8. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DCS Redis高可用实例
4. 运行 `terraform show` 查看已创建的DCS Redis高可用实例

## 参考信息

- [华为云DCS产品文档](https://support.huaweicloud.com/dcs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DCS Redis高可用实例最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-high-availability-instance)
