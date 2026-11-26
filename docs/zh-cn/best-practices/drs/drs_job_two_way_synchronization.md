# 部署双向同步任务

## 应用场景

数据复制服务（Data Replication Service，DRS）是华为云提供的一站式数据复制服务，支持在保证业务连续性的前提下实现数据库之间的实时同步。在业务双活、读写分离、数据汇聚等场景中，往往需要在两个数据库实例之间建立双向的数据同步链路，使两端数据保持一致。

本最佳实践将介绍如何使用Terraform自动化部署两个DRS双向同步任务，包括VPC、子网与安全组的创建，两个RDS MySQL实例及其待同步数据库的创建，以及正向（up）与反向（down）两个DRS同步任务的创建、对象选择和启动。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DRS节点类型（data.huaweicloud_drs_node_types）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/drs_node_types)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [RDS实例（huaweicloud_rds_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_instance)
- [RDS MySQL数据库（huaweicloud_rds_mysql_database）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_mysql_database)
- [DRS任务（huaweicloud_drs_job）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/drs_job)
- [DRS批量对象选择（huaweicloud_drs_batch_select_objects）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/drs_batch_select_objects)
- [DRS任务启动（huaweicloud_drs_start_job）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/drs_start_job)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_rds_instance
        └── huaweicloud_rds_mysql_database
            └── huaweicloud_drs_batch_select_objects
                └── huaweicloud_drs_start_job

data.huaweicloud_drs_node_types
    └── huaweicloud_drs_job
        └── huaweicloud_drs_batch_select_objects
            └── huaweicloud_drs_start_job

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_rds_instance

huaweicloud_networking_secgroup
    └── huaweicloud_networking_secgroup_rule
        └── huaweicloud_rds_instance
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建虚拟私有云和子网

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云和子网资源
variable "vpc_name" {
  description = "The VPC name"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

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

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.gateway_ip
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量 vpc_name 进行赋值
- **cidr**：VPC的CIDR网段，通过引用输入变量 vpc_cidr 进行赋值
- **vpc_id**：子网所属的VPC，引用上一步创建的VPC的ID进行赋值
- **cidr**：子网的CIDR网段，当输入变量 subnet_cidr 为空时，基于VPC网段自动划分子网
- **gateway_ip**：子网网关IP，当输入变量 gateway_ip 为空时，基于子网网段自动计算网关地址

### 3. 创建安全组及其规则

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组及其规则资源
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
- **security_group_id**：规则所属的安全组，引用上一步创建的安全组ID进行赋值
- **direction**：规则方向，第一条为入方向（ingress），第二条为出方向（egress）
- **ports**：入方向规则开放3306端口，用于MySQL数据库访问
- **remote_ip_prefix**：规则生效的远端网段，设置为 192.168.0.0/16

### 4. 查询可用区列表

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区列表数据源
data "huaweicloud_availability_zones" "test" {}
```

**参数说明**：
- 该数据源无需额外参数，用于获取当前region下的可用区列表，供RDS实例选择主备可用区

### 5. 创建两个RDS MySQL实例

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建两个RDS MySQL实例资源
variable "rds1_name" {
  description = "The name of the first RDS instance"
  type        = string
}

variable "rds2_name" {
  description = "The name of the second RDS instance"
  type        = string
}

variable "rds_flavor" {
  description = "The flavor of the RDS instances"
  type        = string
  default     = "rds.mysql.x1.large.2.ha"
}

variable "rds1_fixed_ip" {
  description = "The fixed IP address of the first RDS instance"
  type        = string
  default     = "192.168.0.50"
}

variable "rds2_fixed_ip" {
  description = "The fixed IP address of the second RDS instance"
  type        = string
  default     = "192.168.0.51"
}

variable "db_password" {
  description = "The password for the RDS root user and DRS database connections"
  type        = string
  sensitive   = true
}

resource "huaweicloud_rds_instance" "test" {
  count = 2

  depends_on = [
    huaweicloud_networking_secgroup_rule.test,
  ]

  name                = count.index == 0 ? var.rds1_name : var.rds2_name
  flavor              = var.rds_flavor
  security_group_id   = huaweicloud_networking_secgroup.test.id
  subnet_id           = huaweicloud_vpc_subnet.test.id
  vpc_id              = huaweicloud_vpc.test.id
  fixed_ip            = count.index == 0 ? var.rds1_fixed_ip : var.rds2_fixed_ip
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
- **count**：创建两个RDS实例，索引0为第一个实例，索引1为第二个实例
- **name**：实例名称，索引0引用输入变量 rds1_name，索引1引用输入变量 rds2_name
- **flavor**：实例规格，通过引用输入变量 rds_flavor 进行赋值
- **security_group_id**：实例绑定的安全组，引用前面创建的安全组ID进行赋值
- **subnet_id**：实例所属子网，引用前面创建的子网ID进行赋值
- **vpc_id**：实例所属VPC，引用前面创建的VPC ID进行赋值
- **fixed_ip**：实例的固定IP，索引0引用输入变量 rds1_fixed_ip，索引1引用输入变量 rds2_fixed_ip，均需位于子网网段内
- **ha_replication_mode**：主备实例的复制方式，设置为 semisync（半同步）
- **availability_zone**：实例的主备可用区，引用可用区数据源返回的可用区名称进行赋值
- **db.password**：数据库root用户密码，通过引用输入变量 db_password 进行赋值
- **db.type**：数据库引擎类型，设置为 MySQL
- **db.version**：数据库版本，设置为 5.7
- **db.port**：数据库端口，设置为 3306
- **volume.type**：存储类型，设置为 CLOUDSSD
- **volume.size**：存储大小，设置为 40GB

### 6. 在两个实例上创建待同步数据库

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RDS MySQL数据库资源
variable "db_name" {
  description = "The name of the database to be synchronized in both directions"
  type        = string
  default     = "test"
}

resource "huaweicloud_rds_mysql_database" "test" {
  count = 2

  instance_id   = huaweicloud_rds_instance.test[count.index].id
  name          = var.db_name
  character_set = "utf8"
}
```

**参数说明**：
- **count**：在两个RDS实例上分别创建数据库，使两端均具备可作为源库的待同步数据
- **instance_id**：数据库所属的RDS实例，引用前面创建的RDS实例ID进行赋值
- **name**：数据库名称，通过引用输入变量 db_name 进行赋值
- **character_set**：数据库字符集，设置为 utf8

### 7. 查询DRS任务节点类型

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询DRS节点类型数据源
data "huaweicloud_drs_node_types" "test" {
  count = 2

  engine_type = "mysql"
  type        = "sync"
  direction   = count.index == 0 ? "up" : "down"
}
```

**参数说明**：
- **count**：分别查询正向（up）和反向（down）两种数据方向的节点类型
- **engine_type**：数据库引擎类型，设置为 mysql
- **type**：任务类型，设置为 sync（同步任务）
- **direction**：数据方向，索引0为 up（正向），索引1为 down（反向）

### 8. 创建两个DRS同步任务

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建两个DRS同步任务资源
variable "forward_job_name" {
  description = "The name of the forward DRS job (from the first RDS instance to the second one)"
  type        = string
}

variable "reverse_job_name" {
  description = "The name of the reverse DRS job (from the second RDS instance to the first one)"
  type        = string
}

variable "forward_job_description" {
  description = "The description of the forward DRS job"
  type        = string
  default     = ""
}

variable "reverse_job_description" {
  description = "The description of the reverse DRS job"
  type        = string
  default     = ""
}

variable "db_user" {
  description = "The database user name used by the RDS instances and DRS jobs"
  type        = string
  default     = "root"
}

variable "limit_speed" {
  description = "The speed limit of the DRS jobs. If not specified, the jobs are not limited"
  type = object({
    speed      = string
    start_time = string
    end_time   = string
  })
  default = null
}

resource "huaweicloud_drs_job" "test" {
  count = 2

  name                    = count.index == 0 ? var.forward_job_name : var.reverse_job_name
  type                    = "sync"
  engine_type             = "mysql"
  direction               = count.index == 0 ? "up" : "down"
  node_type               = try(data.huaweicloud_drs_node_types.test[count.index].node_types[0], null)
  net_type                = "vpc"
  migration_type          = "FULL_INCR_TRANS"
  description             = count.index == 0 ? var.forward_job_description : var.reverse_job_description
  force_destroy           = true
  is_start_job            = false
  is_pre_check            = false
  destination_db_readnoly = false
  charging_mode           = "postPaid"

  source_db {
    engine_type = "mysql"
    ip          = huaweicloud_rds_instance.test[count.index].fixed_ip
    port        = 3306
    user        = var.db_user
    password    = var.db_password
    vpc_id      = count.index == 0 ? huaweicloud_rds_instance.test[count.index].vpc_id : null
    instance_id = count.index == 1 ? huaweicloud_rds_instance.test[count.index].id : null
    subnet_id   = huaweicloud_rds_instance.test[count.index].subnet_id
  }

  destination_db {
    region      = count.index == 0 ? huaweicloud_rds_instance.test[1 - count.index].region : null
    ip          = huaweicloud_rds_instance.test[1 - count.index].fixed_ip
    port        = 3306
    engine_type = "mysql"
    user        = var.db_user
    password    = var.db_password
    instance_id = count.index == 0 ? huaweicloud_rds_instance.test[1 - count.index].id : null
    vpc_id      = count.index == 1 ? huaweicloud_rds_instance.test[1 - count.index].vpc_id : null
    subnet_id   = huaweicloud_rds_instance.test[1 - count.index].subnet_id
  }

  policy_config {
    filter_ddl_policy = "drop_database"
    conflict_policy   = "overwrite"
    index_trans       = true
  }

  dynamic "limit_speed" {
    for_each = var.limit_speed == null ? [] : [var.limit_speed]

    content {
      speed      = limit_speed.value.speed
      start_time = limit_speed.value.start_time
      end_time   = limit_speed.value.end_time
    }
  }

  lifecycle {
    ignore_changes = [
      source_db.0.password, destination_db.0.password, force_destroy,
    ]
  }
}
```

**参数说明**：
- **count**：创建两个DRS任务，索引0为正向任务（第一个实例到第二个实例），索引1为反向任务（第二个实例到第一个实例）
- **name**：任务名称，索引0引用输入变量 forward_job_name，索引1引用输入变量 reverse_job_name
- **type**：任务类型，设置为 sync（同步任务）
- **engine_type**：数据库引擎类型，设置为 mysql
- **direction**：数据方向，索引0为 up，索引1为 down
- **node_type**：任务节点类型，引用DRS节点类型数据源返回的节点类型进行赋值
- **net_type**：网络类型，设置为 vpc
- **migration_type**：迁移类型，设置为 FULL_INCR_TRANS（全量+增量）
- **description**：任务描述，索引0引用输入变量 forward_job_description，索引1引用输入变量 reverse_job_description
- **is_start_job**：是否创建后立即启动任务，设置为 false，以便在任务处于配置状态时选择同步对象
- **is_pre_check**：是否执行预检查，设置为 false
- **charging_mode**：计费模式，设置为 postPaid（按需计费）
- **source_db**：源数据库信息，索引0以第一个实例为源，索引1以第二个实例为源，包含引擎类型、IP、端口、用户名、密码、VPC、实例ID和子网
- **destination_db**：目标数据库信息，索引0以第二个实例为目标，索引1以第一个实例为目标，包含区域、IP、端口、引擎类型、用户名、密码、实例ID、VPC和子网
- **policy_config**：同步策略配置，包括DDL过滤策略、冲突策略和索引传输开关
- **limit_speed**：限速配置，当输入变量 limit_speed 不为空时生效

### 9. 为任务选择同步对象

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下为DRS任务选择同步对象
resource "huaweicloud_drs_batch_select_objects" "test" {
  count = 2

  jobs {
    job_id        = huaweicloud_drs_job.test[count.index].id
    selected      = true
    sync_database = true

    job {
      id          = huaweicloud_rds_mysql_database.test[count.index].name
      object_type = "database"
      object_name = huaweicloud_rds_mysql_database.test[count.index].name
      select      = "true"
    }
  }
}
```

**参数说明**：
- **count**：分别为正向和反向两个任务选择同步对象
- **job_id**：待选择对象的DRS任务，引用前面创建的DRS任务ID进行赋值
- **selected**：是否已选择对象，设置为 true
- **sync_database**：是否同步数据库，设置为 true
- **job.id**：同步对象ID，引用前面创建的数据库名称进行赋值
- **job.object_type**：对象类型，设置为 database
- **job.object_name**：对象名称，引用前面创建的数据库名称进行赋值
- **job.select**：是否选中该对象，设置为 true

### 10. 启动DRS同步任务

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下启动DRS同步任务
resource "huaweicloud_drs_start_job" "test" {
  count = 2

  depends_on = [huaweicloud_drs_batch_select_objects.test]

  job_id = huaweicloud_drs_job.test[count.index].id
}
```

**参数说明**：
- **count**：分别启动正向和反向两个DRS任务
- **depends_on**：确保在同步对象选择完成后再启动任务
- **job_id**：待启动的DRS任务，引用前面创建的DRS任务ID进行赋值

### 11. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
vpc_name            = "tf_test_drs_vpc"
subnet_name         = "tf_test_drs_subnet"
security_group_name = "tf_test_drs_sg"
rds1_name           = "tf_test_drs_rds1"
rds2_name           = "tf_test_drs_rds2"
rds_flavor          = "rds.mysql.x1.large.2.ha"
rds1_fixed_ip       = "192.168.0.50"
rds2_fixed_ip       = "192.168.0.51"
db_password         = "<your_db_password>"
db_name             = "test"
forward_job_name    = "tf_test_forward_drs_job"
reverse_job_name    = "tf_test_reverse_drs_job"
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

### 12. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建双向同步任务
4. 运行 `terraform show` 查看已创建的双向同步任务

## 参考信息

- [华为云数据复制服务产品文档](https://support.huaweicloud.com/drs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DRS双向同步任务最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/drs/drs-job-two-way-synchronization)
