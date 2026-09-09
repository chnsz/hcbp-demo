# 部署Redis数据同步

## 应用场景

分布式缓存服务（Distributed Cache Service, DCS）是华为云提供的高性能、高可用的内存数据库服务，支持Redis、Memcached等主流缓存引擎。在业务迁移、容灾或数据整合等场景中，经常需要将数据从一个Redis实例同步到另一个实例。

本最佳实践将介绍如何使用Terraform自动化部署两个DCS Redis实例，并创建全量及增量在线数据迁移任务，实现源实例到目标实例的数据同步。通过本实践，您可以快速搭建一套完整的Redis数据同步环境，包括VPC、子网、安全组、DCS实例及迁移任务。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用分区（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DCS产品规格（data.huaweicloud_dcs_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dcs_flavors)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [DCS实例（huaweicloud_dcs_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_instance)
- [DCS在线数据迁移任务（huaweicloud_dcs_online_data_migration_task）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dcs_online_data_migration_task)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_dcs_instance

data.huaweicloud_dcs_flavors
    └── huaweicloud_dcs_instance

huaweicloud_vpc
    ├── huaweicloud_vpc_subnet
    └── huaweicloud_dcs_instance

huaweicloud_vpc_subnet
    └── huaweicloud_dcs_instance

huaweicloud_networking_secgroup
    └── huaweicloud_dcs_instance

huaweicloud_dcs_instance
    └── huaweicloud_dcs_online_data_migration_task
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建VPC和子网

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC和子网
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
  default     = "dcs-sync-vpc"
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
  default     = "dcs-sync-subnet"
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  default     = ""
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**参数说明**：
- **name**：通过引用输入变量 vpc_name 进行赋值，指定VPC名称。
- **cidr**：通过引用输入变量 vpc_cidr 进行赋值，指定VPC的CIDR块。
- **vpc_id**：通过引用 huaweicloud_vpc.test.id 进行赋值，指定子网所属的VPC。
- **name**：通过引用输入变量 subnet_name 进行赋值，指定子网名称。
- **cidr**：通过引用输入变量 subnet_cidr 进行赋值，若未设置则自动从VPC的CIDR中划分。
- **gateway_ip**：通过引用输入变量 subnet_gateway_ip 进行赋值，若未设置则自动计算网关地址。

### 3. 创建安全组

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
  default     = "dcs-sync-sg"
}

resource "huaweicloud_networking_secgroup" "test" {
  name        = var.security_group_name
  description = "Security group for DCS data migration"
}
```

**参数说明**：
- **name**：通过引用输入变量 security_group_name 进行赋值，指定安全组名称。
- **description**：安全组描述信息。

### 4. 查询可用分区和DCS产品规格

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用分区和DCS产品规格
data "huaweicloud_availability_zones" "test" {}

data "huaweicloud_dcs_flavors" "test" {
  cache_mode     = var.instance_cache_mode
  capacity       = var.instance_capacity
  engine_version = var.instance_engine_version
}
```

**参数说明**：
- **cache_mode**：通过引用输入变量 instance_cache_mode 进行赋值，指定缓存类型。
- **capacity**：通过引用输入变量 instance_capacity 进行赋值，指定缓存容量（GB）。
- **engine_version**：通过引用输入变量 instance_engine_version 进行赋值，指定引擎版本。

### 5. 创建源和目标DCS Redis实例

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建源和目标DCS Redis实例
variable "instance_cache_mode" {
  description = "The cache mode of the DCS instances"
  type        = string
  default     = "ha"
}

variable "instance_capacity" {
  description = "The capacity of the DCS instances (GB)"
  type        = number
  default     = 4
}

variable "instance_engine_version" {
  description = "The engine version of the DCS instances"
  type        = string
  default     = "5.0"
}

variable "instance_name" {
  description = "The base name of the DCS Redis instances (will be suffixed with -0 and -1)"
  type        = string
}

variable "instance_password" {
  description = "The password of the DCS instances"
  type        = string
  sensitive   = true
}

resource "huaweicloud_dcs_instance" "test" {
  count = 2

  name               = "${var.instance_name}-${count.index}"
  engine             = "Redis"
  engine_version     = var.instance_engine_version
  capacity           = var.instance_capacity
  flavor             = try(data.huaweicloud_dcs_flavors.test.flavors[0].name, null)
  availability_zones = try(slice(data.huaweicloud_availability_zones.test.names, 0, 2), null)
  vpc_id             = huaweicloud_vpc.test.id
  subnet_id          = huaweicloud_vpc_subnet.test.id
  security_group_id  = huaweicloud_networking_secgroup.test.id
  password           = var.instance_password

  lifecycle {
    ignore_changes = [
      security_group_id
    ]
  }
}
```

**参数说明**：
- **count**：创建2个实例，分别作为源实例和目标实例。
- **name**：通过引用输入变量 instance_name 和 count.index 进行赋值，实例名称分别为 `${instance_name}-0` 和 `${instance_name}-1`。
- **engine**：缓存引擎，固定为Redis。
- **engine_version**：通过引用输入变量 instance_engine_version 进行赋值，指定引擎版本。
- **capacity**：通过引用输入变量 instance_capacity 进行赋值，指定缓存容量（GB）。
- **flavor**：通过引用 data.huaweicloud_dcs_flavors.test.flavors[0].name 进行赋值，指定实例规格。
- **availability_zones**：通过引用 data.huaweicloud_availability_zones.test.names 进行赋值，指定可用分区。
- **vpc_id**：通过引用 huaweicloud_vpc.test.id 进行赋值，指定实例所属VPC。
- **subnet_id**：通过引用 huaweicloud_vpc_subnet.test.id 进行赋值，指定实例所属子网。
- **security_group_id**：通过引用 huaweicloud_networking_secgroup.test.id 进行赋值，指定安全组。
- **password**：通过引用输入变量 instance_password 进行赋值，指定实例密码。

### 6. 创建全量迁移任务

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建全量迁移任务
variable "full_migration_task_name" {
  description = "The name of the full migration task"
  type        = string
  default     = "full-migration-task"
}

variable "full_migration_task_description" {
  description = "The description of the full migration task"
  type        = string
  default     = "Full data migration from source to target DCS instance"
}

variable "full_migration_resume_mode" {
  description = "The reconnection mode for full migration"
  type        = string
  default     = "auto"
}

variable "full_migration_bandwidth_limit_mb" {
  description = "The bandwidth limit for full migration (MB/s)"
  type        = string
  default     = ""
}

resource "huaweicloud_dcs_online_data_migration_task" "full_migration" {
  task_name          = var.full_migration_task_name
  vpc_id             = huaweicloud_vpc.test.id
  subnet_id          = huaweicloud_vpc_subnet.test.id
  security_group_id  = huaweicloud_networking_secgroup.test.id
  description        = var.full_migration_task_description
  migration_method   = "full_amount_migration"
  resume_mode        = var.full_migration_resume_mode
  bandwidth_limit_mb = var.full_migration_bandwidth_limit_mb != "" ? var.full_migration_bandwidth_limit_mb : null

  source_instance {
    id       = huaweicloud_dcs_instance.test[0].id
    password = var.instance_password
  }

  target_instance {
    id       = huaweicloud_dcs_instance.test[1].id
    password = var.instance_password
  }

  lifecycle {
    ignore_changes = [
      source_instance,
      target_instance
    ]
  }

  depends_on = [
    huaweicloud_dcs_instance.test
  ]
}
```

**参数说明**：
- **task_name**：通过引用输入变量 full_migration_task_name 进行赋值，指定迁移任务名称。
- **vpc_id**：通过引用 huaweicloud_vpc.test.id 进行赋值，指定迁移任务所属VPC。
- **subnet_id**：通过引用 huaweicloud_vpc_subnet.test.id 进行赋值，指定迁移任务所属子网。
- **security_group_id**：通过引用 huaweicloud_networking_secgroup.test.id 进行赋值，指定安全组。
- **description**：通过引用输入变量 full_migration_task_description 进行赋值，指定任务描述。
- **migration_method**：迁移方式，固定为`full_amount_migration`，表示全量迁移。
- **resume_mode**：通过引用输入变量 full_migration_resume_mode 进行赋值，指定断点续传模式。
- **bandwidth_limit_mb**：通过引用输入变量 full_migration_bandwidth_limit_mb 进行赋值，指定带宽限制，为空时不限制。
- **source_instance**：源实例配置，通过引用 huaweicloud_dcs_instance.test[0].id 和输入变量 instance_password 进行赋值。
- **target_instance**：目标实例配置，通过引用 huaweicloud_dcs_instance.test[1].id 和输入变量 instance_password 进行赋值。

### 7. 创建增量迁移任务（可选）

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建增量迁移任务
variable "enable_incremental_migration" {
  description = "Whether to enable incremental migration after full migration"
  type        = bool
  default     = true
}

variable "incremental_migration_task_name" {
  description = "The name of the incremental migration task"
  type        = string
  default     = "incremental-migration-task"
}

variable "incremental_migration_task_description" {
  description = "The description of the incremental migration task"
  type        = string
  default     = "Incremental data migration from source to target DCS instance"
}

variable "incremental_migration_resume_mode" {
  description = "The reconnection mode for incremental migration"
  type        = string
  default     = "auto"
}

variable "incremental_migration_bandwidth_limit_mb" {
  description = "The bandwidth limit for incremental migration (MB/s)"
  type        = string
  default     = ""
}

resource "huaweicloud_dcs_online_data_migration_task" "incremental_migration" {
  count = var.enable_incremental_migration ? 1 : 0

  task_name          = var.incremental_migration_task_name
  vpc_id             = huaweicloud_vpc.test.id
  subnet_id          = huaweicloud_vpc_subnet.test.id
  security_group_id  = huaweicloud_networking_secgroup.test.id
  description        = var.incremental_migration_task_description
  migration_method   = "incremental_migration"
  resume_mode        = var.incremental_migration_resume_mode
  bandwidth_limit_mb = var.incremental_migration_bandwidth_limit_mb != "" ? var.incremental_migration_bandwidth_limit_mb : null

  source_instance {
    id       = huaweicloud_dcs_instance.test[0].id
    password = var.instance_password
  }

  target_instance {
    id       = huaweicloud_dcs_instance.test[1].id
    password = var.instance_password
  }

  lifecycle {
    ignore_changes = [
      source_instance,
      target_instance
    ]
  }

  depends_on = [
    huaweicloud_dcs_instance.test,
    huaweicloud_dcs_online_data_migration_task.full_migration
  ]
}
```

**参数说明**：
- **count**：通过引用输入变量 enable_incremental_migration 进行赋值，当为true时创建1个增量迁移任务，否则不创建。
- **task_name**：通过引用输入变量 incremental_migration_task_name 进行赋值，指定迁移任务名称。
- **vpc_id**：通过引用 huaweicloud_vpc.test.id 进行赋值，指定迁移任务所属VPC。
- **subnet_id**：通过引用 huaweicloud_vpc_subnet.test.id 进行赋值，指定迁移任务所属子网。
- **security_group_id**：通过引用 huaweicloud_networking_secgroup.test.id 进行赋值，指定安全组。
- **description**：通过引用输入变量 incremental_migration_task_description 进行赋值，指定任务描述。
- **migration_method**：迁移方式，固定为`incremental_migration`，表示增量迁移。
- **resume_mode**：通过引用输入变量 incremental_migration_resume_mode 进行赋值，指定断点续传模式。
- **bandwidth_limit_mb**：通过引用输入变量 incremental_migration_bandwidth_limit_mb 进行赋值，指定带宽限制，为空时不限制。
- **source_instance**：源实例配置，通过引用 huaweicloud_dcs_instance.test[0].id 和输入变量 instance_password 进行赋值。
- **target_instance**：目标实例配置，通过引用 huaweicloud_dcs_instance.test[1].id 和输入变量 instance_password 进行赋值。

### 8. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# DCS Instance Configuration
instance_name     = "redis-instance"
instance_password = "YourPassword@123"

# Full Migration Task Configuration
full_migration_bandwidth_limit_mb = "100"

# Incremental Migration Task Configuration
enable_incremental_migration             = true
incremental_migration_bandwidth_limit_mb = "50"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="instance_name=my-instance"`
2. 环境变量：`export TF_VAR_instance_name=my-instance`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 9. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DCS Redis实例及迁移任务
4. 运行 `terraform show` 查看已创建的DCS Redis实例及迁移任务

## 参考信息

- [华为云DCS产品文档](https://support.huaweicloud.com/dcs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DCS Redis数据同步最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dcs/redis-data-sync)
