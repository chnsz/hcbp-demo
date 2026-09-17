# 部署迁移任务

## 应用场景

数据复制服务（Data Replication Service，DRS）是华为云提供的一站式数据复制服务，支持在保证业务连续性的前提下，实现数据在云上、云下以及跨云环境之间的高效复制。在数据库上云、数据库迁移等场景中，通常需要将源数据库的数据全量加增量地迁移至目标数据库，并保持源端业务不中断。

本最佳实践将介绍如何使用Terraform自动化部署一个DRS迁移任务，包括VPC、子网、安全组、源端与目标端RDS MySQL实例的创建，以及DRS迁移任务的配置。通过本实践，您可以快速掌握使用Terraform编排DRS迁移任务的方法，为后续的数据库迁移与运维工作奠定基础。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [RDS规格列表（data.huaweicloud_rds_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/rds_flavors)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [RDS实例（huaweicloud_rds_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_instance)
- [DRS任务（huaweicloud_drs_job）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/drs_job)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_rds_instance

data.huaweicloud_rds_flavors
    └── huaweicloud_rds_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_rds_instance

huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule
    └── huaweicloud_rds_instance

huaweicloud_rds_instance
    └── huaweicloud_drs_job
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建虚拟私有云

在TF文件（如main.tf）中添加以下脚本以创建虚拟私有云：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云资源
variable "vpc_name" {
  description = "The VPC name"
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
- **cidr**：VPC网段，通过引用输入变量 vpc_cidr 进行赋值

### 3. 创建虚拟私有云子网

在TF文件（如main.tf）中添加以下脚本以创建子网：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云子网资源
variable "subnet_name" {
  description = "The subnet name"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

variable "gateway_ip" {
  description = "The gateway IP address of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.gateway_ip
}
```

**参数说明**：
- **vpc_id**：子网所属的VPC ID，引用前一步创建的VPC资源的ID
- **name**：子网名称，通过引用输入变量 subnet_name 进行赋值
- **cidr**：子网网段，通过引用输入变量 subnet_cidr 进行赋值；当取值为空字符串时，基于VPC网段自动划分子网
- **gateway_ip**：子网网关IP，通过引用输入变量 gateway_ip 进行赋值；当取值为空字符串时，基于子网网段自动计算网关IP

### 4. 创建安全组及其规则

在TF文件（如main.tf）中添加以下脚本以创建安全组及安全组规则：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源
variable "security_group_name" {
  description = "The security group name"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}

resource "huaweicloud_networking_secgroup_rule" "test" {
  count = 2

  security_group_id = huaweicloud_networking_secgroup.test.id
  ethertype         = "IPv4"
  remote_ip_prefix  = "192.168.0.0/16"
  protocol          = "tcp"
  direction         = count.index == 0 ? "ingress" : "egress"
  ports             = count.index == 0 ? "3306" : null
}
```

**参数说明**：
- **name**：安全组名称，通过引用输入变量 security_group_name 进行赋值
- **delete_default_rules**：是否删除安全组默认规则，设置为 true 以便仅保留自定义规则
- **security_group_id**：安全组规则所属的安全组ID，引用前一步创建的安全组资源的ID
- **ethertype**：网络类型，设置为 IPv4
- **remote_ip_prefix**：远端IP地址范围，设置为 192.168.0.0/16
- **protocol**：协议类型，设置为 tcp
- **direction**：规则方向，第一条规则为 ingress（入方向），第二条规则为 egress（出方向）
- **ports**：端口范围，入方向规则开放 3306 端口，出方向规则不限制端口

### 5. 查询可用区与RDS规格

在TF文件（如main.tf）中添加以下脚本以查询可用区列表和RDS规格列表：

```hcl
# 查询当前region下的可用区列表
data "huaweicloud_availability_zones" "test" {}

# 查询RDS规格列表
variable "rds_db_type" {
  description = "The database type for querying RDS flavors"
  type        = string
  default     = "MySQL"
}

variable "rds_db_version" {
  description = "The database version for querying RDS flavors"
  type        = string
  default     = "5.7"
}

variable "rds_instance_mode" {
  description = "The instance mode for querying RDS flavors"
  type        = string
  default     = "ha"
}

data "huaweicloud_rds_flavors" "test" {
  db_type       = var.rds_db_type
  db_version    = var.rds_db_version
  instance_mode = var.rds_instance_mode
}
```

**参数说明**：
- **db_type**：数据库类型，通过引用输入变量 rds_db_type 进行赋值
- **db_version**：数据库版本，通过引用输入变量 rds_db_version 进行赋值
- **instance_mode**：实例模式，通过引用输入变量 rds_instance_mode 进行赋值

### 6. 创建源端与目标端RDS实例

在TF文件（如main.tf）中添加以下脚本以创建源端和目标端RDS MySQL实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RDS实例资源
variable "source_rds_name" {
  description = "The name of the source RDS instance"
  type        = string
}

variable "dest_rds_name" {
  description = "The name of the destination RDS instance"
  type        = string
}

variable "rds_flavor" {
  description = "The flavor of the RDS instances"
  type        = string
  default     = "rds.mysql.x1.large.2.ha"
}

variable "source_rds_fixed_ip" {
  description = "The fixed IP address of the source RDS instance"
  type        = string
}

variable "dest_rds_fixed_ip" {
  description = "The fixed IP address of the destination RDS instance"
  type        = string
}

variable "db_password" {
  description = "The password for the RDS root user and DRS database connections"
  type        = string
  sensitive   = true
}

resource "huaweicloud_rds_instance" "test" {
  count = 2

  name                = count.index == 0 ? var.source_rds_name : var.dest_rds_name
  flavor              = var.rds_flavor != "" ? var.rds_flavor : try(data.huaweicloud_rds_flavors.test.flavors[0].name, null)
  security_group_id   = huaweicloud_networking_secgroup.test.id
  subnet_id           = huaweicloud_vpc_subnet.test.id
  vpc_id              = huaweicloud_vpc.test.id
  fixed_ip            = count.index == 0 ? var.source_rds_fixed_ip : var.dest_rds_fixed_ip
  ha_replication_mode = "semisync"

  availability_zone = [
    try(data.huaweicloud_availability_zones.test.names[0], ""),
    try(data.huaweicloud_availability_zones.test.names[3], ""),
  ]

  db {
    password = var.db_password
    type     = "MySQL"
    version  = "5.7"
    port     = 3306
  }

  volume {
    type = "CLOUDSSD"
    size = 40
  }
}
```

**参数说明**：
- **count**：实例数量，设置为 2，分别对应源端实例（索引 0）和目标端实例（索引 1）
- **name**：实例名称，源端引用输入变量 source_rds_name，目标端引用输入变量 dest_rds_name
- **flavor**：实例规格，通过引用输入变量 rds_flavor 进行赋值；当取值为空字符串时，使用数据源查询到的第一个规格
- **security_group_id**：实例所属的安全组ID，引用前一步创建的安全组资源的ID
- **subnet_id**：实例所属的子网ID，引用前一步创建的子网资源的ID
- **vpc_id**：实例所属的VPC ID，引用前一步创建的VPC资源的ID
- **fixed_ip**：实例的固定IP地址，源端引用输入变量 source_rds_fixed_ip，目标端引用输入变量 dest_rds_fixed_ip
- **ha_replication_mode**：主备复制模式，设置为 semisync（半同步）
- **availability_zone**：实例的可用区列表，引用可用区数据源查询结果
- **db**：数据库配置，包括密码（引用输入变量 db_password）、类型（MySQL）、版本（5.7）和端口（3306）
- **volume**：存储配置，包括存储类型（CLOUDSSD）和存储大小（40GB）

### 7. 创建DRS迁移任务

在TF文件（如main.tf）中添加以下脚本以创建DRS迁移任务：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DRS任务资源
variable "job_name" {
  description = "The DRS job name"
  type        = string
}

variable "description" {
  description = "The description of the DRS job"
  type        = string
  default     = ""
}

variable "tags" {
  description = "The tags of the DRS job"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_drs_job" "test" {
  name           = var.job_name
  type           = "migration"
  engine_type    = "mysql"
  direction      = "up"
  net_type       = "eip"
  migration_type = "FULL_INCR_TRANS"
  description    = var.description
  force_destroy  = true

  source_db {
    engine_type = "mysql"
    ip          = huaweicloud_rds_instance.test[0].fixed_ip
    port        = 3306
    user        = "root"
    password    = var.db_password
    ssl_enabled = false
  }

  destination_db {
    region      = huaweicloud_rds_instance.test[1].region
    ip          = huaweicloud_rds_instance.test[1].fixed_ip
    port        = 3306
    engine_type = "mysql"
    user        = "root"
    password    = var.db_password
    instance_id = huaweicloud_rds_instance.test[1].id
    subnet_id   = huaweicloud_rds_instance.test[1].subnet_id
  }

  tags = var.tags

  lifecycle {
    ignore_changes = [
      source_db.0.password, destination_db.0.password, force_destroy, action,
    ]
  }
}
```

**参数说明**：
- **name**：DRS任务名称，通过引用输入变量 job_name 进行赋值
- **type**：任务类型，设置为 migration（迁移）
- **engine_type**：数据库引擎类型，设置为 mysql
- **direction**：迁移方向，设置为 up（上云）
- **net_type**：网络类型，设置为 eip
- **migration_type**：迁移模式，设置为 FULL_INCR_TRANS（全量加增量）
- **description**：任务描述，通过引用输入变量 description 进行赋值
- **force_destroy**：是否强制删除，设置为 true
- **source_db**：源数据库配置，包括引擎类型（mysql）、IP地址（引用源端RDS实例的固定IP）、端口（3306）、用户名（root）、密码（引用输入变量 db_password）和SSL开关（false）
- **destination_db**：目标数据库配置，包括区域（引用目标端RDS实例的区域）、IP地址（引用目标端RDS实例的固定IP）、端口（3306）、引擎类型（mysql）、用户名（root）、密码（引用输入变量 db_password）、实例ID（引用目标端RDS实例的ID）和子网ID（引用目标端RDS实例的子网ID）
- **tags**：任务标签，通过引用输入变量 tags 进行赋值
- **lifecycle**：生命周期配置，忽略 source_db.0.password、destination_db.0.password、force_destroy 和 action 属性的变更

### 8. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# VPC与网络变量
vpc_name            = "your_vpc"
subnet_name         = "your_subnet"
security_group_name = "your_security_group"

# RDS实例变量
source_rds_name     = "your_source_rds"
dest_rds_name       = "your_dest_rds"
rds_flavor          = "rds.mysql.x1.large.2.ha"
source_rds_fixed_ip = "192.168.0.58"
dest_rds_fixed_ip   = "192.168.0.59"
db_password         = "TestDrs@123"

# DRS任务变量
job_name = "your_drs_job"
tags = {
  foo = "bar"
  key = "value"
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

### 9. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DRS迁移任务
4. 运行 `terraform show` 查看已创建的DRS迁移任务

## 参考信息

- [华为云数据复制服务产品文档](https://support.huaweicloud.com/drs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DRS迁移任务最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/drs/drs-job-migration)
