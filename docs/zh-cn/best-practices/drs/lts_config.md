# 部署LTS配置

## 应用场景

数据复制服务（Data Replication Service，DRS）是华为云提供的一站式数据复制服务，支持数据库上云、数据库迁移、数据库实时同步和数据库灾备等场景。在迁移或同步任务运行过程中，任务的运行日志、错误日志和性能指标对于问题定位和运维监控至关重要。

本最佳实践将介绍如何使用Terraform为DRS任务配置云日志服务（LTS），将DRS任务的运行日志采集到LTS日志组和日志流中，实现日志的集中存储、查询与分析。

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
- [云数据库RDS实例（huaweicloud_rds_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_instance)
- [DRS任务（huaweicloud_drs_job）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/drs_job)
- [LTS日志组（huaweicloud_lts_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [LTS日志流（huaweicloud_lts_stream）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)
- [DRS任务LTS配置（huaweicloud_drs_lts_config）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/drs_lts_config)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_rds_instance

data.huaweicloud_rds_flavors
    └── huaweicloud_rds_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_rds_instance
            └── huaweicloud_drs_job
                └── huaweicloud_drs_lts_config

huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule
    └── huaweicloud_rds_instance

huaweicloud_lts_group
    └── huaweicloud_lts_stream
        └── huaweicloud_drs_lts_config
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建VPC与子网

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC与子网资源
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
- **cidr**：VPC网段，通过引用输入变量 vpc_cidr 进行赋值
- **vpc_id**：子网所属的VPC ID，引用VPC资源的ID进行赋值
- **subnet cidr**：子网网段，当输入变量 subnet_cidr 为空时，通过 cidrsubnet 函数从VPC网段自动划分
- **gateway_ip**：子网网关IP，当输入变量 gateway_ip 为空时，通过 cidrhost 函数自动推导

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
- **delete_default_rules**：是否删除安全组默认规则，设置为 true
- **security_group_id**：规则所属的安全组 ID，引用安全组资源的ID进行赋值
- **direction**：规则方向，第一条为入方向（ingress），第二条为出方向（egress）
- **ports**：入方向规则开放3306端口，用于MySQL数据库访问

### 4. 查询可用区与RDS规格

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区与RDS规格数据源
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

data "huaweicloud_availability_zones" "test" {}

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

### 5. 创建源端与目标端RDS实例

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建源端与目标端RDS实例资源
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
- **count**：创建两个RDS实例，索引0为源端实例，索引1为目标端实例
- **name**：实例名称，源端引用输入变量 source_rds_name，目标端引用输入变量 dest_rds_name
- **flavor**：实例规格，通过引用输入变量 rds_flavor 进行赋值，为空时自动使用查询到的规格
- **security_group_id**：安全组 ID，引用安全组资源的ID进行赋值
- **subnet_id**：子网 ID，引用子网资源的ID进行赋值
- **vpc_id**：VPC ID，引用VPC资源的ID进行赋值
- **fixed_ip**：实例内网IP，源端引用输入变量 source_rds_fixed_ip，目标端引用输入变量 dest_rds_fixed_ip
- **ha_replication_mode**：主备复制模式，设置为 semisync（半同步）
- **availability_zone**：可用区列表，引用可用区数据源进行赋值
- **db.password**：数据库密码，通过引用输入变量 db_password 进行赋值
- **volume**：存储配置，类型为 CLOUDSSD，大小为40GB

### 6. 创建DRS迁移任务

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DRS迁移任务资源
variable "job_name" {
  description = "The DRS job name"
  type        = string
}

variable "description" {
  description = "The description of the DRS job"
  type        = string
  default     = ""
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

  lifecycle {
    ignore_changes = [
      source_db.0.password, destination_db.0.password, force_destroy, action,
    ]
  }
}
```

**参数说明**：
- **name**：任务名称，通过引用输入变量 job_name 进行赋值
- **type**：任务类型，设置为 migration（迁移）
- **engine_type**：引擎类型，设置为 mysql
- **direction**：迁移方向，设置为 up（上云）
- **net_type**：网络类型，设置为 eip
- **migration_type**：迁移模式，设置为 FULL_INCR_TRANS（全量+增量）
- **description**：任务描述，通过引用输入变量 description 进行赋值
- **force_destroy**：是否强制删除，设置为 true，允许任务运行中删除
- **source_db**：源数据库信息，IP引用源端RDS实例的内网IP，密码通过引用输入变量 db_password 进行赋值
- **destination_db**：目标数据库信息，引用目标端RDS实例的ID、内网IP、子网ID等属性进行赋值
- **lifecycle.ignore_changes**：忽略密码、force_destroy 和 action 字段的变更

### 7. 创建LTS日志组与日志流

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建LTS日志组与日志流资源
variable "lts_group_name" {
  description = "The name of the LTS log group"
  type        = string
}

variable "lts_ttl_in_days" {
  description = "The log retention period in days"
  type        = number
  default     = 30
}

variable "lts_stream_name" {
  description = "The name of the LTS log stream"
  type        = string
}

resource "huaweicloud_lts_group" "test" {
  group_name  = var.lts_group_name
  ttl_in_days = var.lts_ttl_in_days
}

resource "huaweicloud_lts_stream" "test" {
  group_id    = huaweicloud_lts_group.test.id
  stream_name = var.lts_stream_name
  is_favorite = true
}
```

**参数说明**：
- **group_name**：日志组名称，通过引用输入变量 lts_group_name 进行赋值
- **ttl_in_days**：日志保存天数，通过引用输入变量 lts_ttl_in_days 进行赋值
- **group_id**：日志流所属的日志组 ID，引用日志组资源的ID进行赋值
- **stream_name**：日志流名称，通过引用输入变量 lts_stream_name 进行赋值
- **is_favorite**：是否收藏日志流，设置为 true

### 8. 创建DRS任务LTS配置

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DRS任务LTS配置资源
resource "huaweicloud_drs_lts_config" "test" {
  job_id        = huaweicloud_drs_job.test.id
  log_group_id  = huaweicloud_lts_group.test.id
  log_stream_id = huaweicloud_lts_stream.test.id
}
```

**参数说明**：
- **job_id**：DRS任务 ID，引用DRS任务资源的ID进行赋值
- **log_group_id**：日志组 ID，引用LTS日志组资源的ID进行赋值
- **log_stream_id**：日志流 ID，引用LTS日志流资源的ID进行赋值

### 9. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
vpc_name            = "drs-vpc"
subnet_name         = "drs-subnet"
security_group_name = "drs-secgroup"
source_rds_name     = "drs-source-rds"
dest_rds_name       = "drs-dest-rds"
source_rds_fixed_ip = "192.168.0.10"
dest_rds_fixed_ip   = "192.168.0.11"
db_password         = "your-strong-password"
job_name            = "drs-migration-job"
lts_group_name      = "drs-lts-group"
lts_stream_name     = "drs-lts-stream"
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

### 10. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DRS任务LTS配置
4. 运行 `terraform show` 查看已创建的DRS任务LTS配置

## 参考信息

- [华为云数据复制服务产品文档](https://support.huaweicloud.com/drs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DRS任务LTS配置最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/drs/lts-config)
