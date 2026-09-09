# 部署DDS实例关联LTS

## 应用场景

文档数据库服务（Document Database Service，DDS）是华为云提供的高性能、高可靠、高安全性的分布式文档数据库服务，完全兼容MongoDB协议。在实际业务中，需要对DDS实例的审计日志进行集中管理和分析，以满足安全合规和运维监控的需求。

本最佳实践将介绍如何使用Terraform创建DDS实例并关联LTS日志组和日志流，实现审计日志的采集与存储。通过本实践，您可以了解如何利用Terraform自动化部署DDS实例、VPC、安全组及LTS资源，并将DDS实例的审计日志接入LTS，为后续的日志分析和告警提供基础。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用分区（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DDS规格（data.huaweicloud_dds_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dds_flavors)

### 资源

- [虚拟私有云VPC（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [LTS日志组（huaweicloud_lts_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_group)
- [LTS日志流（huaweicloud_lts_stream）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/lts_stream)
- [DDS实例（huaweicloud_dds_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dds_instance)
- [DDS LTS日志关联（huaweicloud_dds_lts_log）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dds_lts_log)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_dds_instance
huaweicloud_vpc
    ├── huaweicloud_vpc_subnet
    └── huaweicloud_dds_instance
huaweicloud_vpc_subnet
    └── huaweicloud_dds_instance
huaweicloud_networking_secgroup
    └── huaweicloud_dds_instance
data.huaweicloud_dds_flavors
    └── huaweicloud_dds_instance
huaweicloud_lts_group
    └── huaweicloud_lts_stream
huaweicloud_lts_stream
    └── huaweicloud_dds_lts_log
huaweicloud_dds_instance
    └── huaweicloud_dds_lts_log
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建VPC

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC
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
- **name**：通过引用输入变量 vpc_name 进行赋值。
- **cidr**：通过引用输入变量 vpc_cidr 进行赋值。

### 3. 创建子网

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建子网
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
  description = "The gateway IP of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**参数说明**：
- **vpc_id**：通过引用 huaweicloud_vpc.test.id 进行赋值。
- **name**：通过引用输入变量 subnet_name 进行赋值。
- **cidr**：通过引用输入变量 subnet_cidr 进行赋值，当 subnet_cidr 为空时，自动从VPC的CIDR中划分。
- **gateway_ip**：通过引用输入变量 subnet_gateway_ip 进行赋值，当 subnet_gateway_ip 为空时，自动计算子网的网关地址。

### 4. 创建安全组

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：通过引用输入变量 security_group_name 进行赋值。
- **delete_default_rules**：是否删除默认规则，设置为 true 表示删除。

### 5. 创建LTS日志组和日志流

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建LTS日志组和日志流
variable "group_name" {
  description = "The name of the LTS log group"
  type        = string
}

variable "group_log_expiration_days" {
  description = "The log expiration time of the LTS log group"
  type        = number
  default     = 30
  nullable    = false
}

variable "stream_name" {
  description = "The name of the LTS log stream"
  type        = string
}

resource "huaweicloud_lts_group" "test" {
  group_name  = var.group_name
  ttl_in_days = var.group_log_expiration_days
}

resource "huaweicloud_lts_stream" "test" {
  group_id    = huaweicloud_lts_group.test.id
  stream_name = var.stream_name
}
```

**参数说明**：
- **huaweicloud_lts_group.test**：
  - **group_name**：通过引用输入变量 group_name 进行赋值。
  - **ttl_in_days**：通过引用输入变量 group_log_expiration_days 进行赋值，用于设置日志保存时间。
- **huaweicloud_lts_stream.test**：
  - **group_id**：通过引用 huaweicloud_lts_group.test.id 进行赋值。
  - **stream_name**：通过引用输入变量 stream_name 进行赋值。

### 6. 查询可用分区和DDS规格

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用分区和DDS规格
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}

data "huaweicloud_dds_flavors" "test" {
  count = var.node_spec_code == "" ? 1 : 0

  engine_name = var.engine_name
  vcpus       = var.flavor_vcpus
  memory      = var.flavor_memory
  type        = var.node_type
}
```

**参数说明**：
- **data.huaweicloud_availability_zones.test**：当 availability_zone 为空时，查询可用分区列表，用于自动选择可用分区。
- **data.huaweicloud_dds_flavors.test**：当 node_spec_code 为空时，根据引擎名称、vCPU、内存和节点类型查询DDS规格，用于自动选择规格。

### 7. 创建DDS实例

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DDS实例
variable "availability_zone" {
  description = "The availability zone to which the DDS instance belongs"
  type        = string
  default     = ""
  nullable    = false
}

variable "node_spec_code" {
  description = "The node specification code of the DDS instance"
  type        = string
  default     = ""
  nullable    = false
}

variable "engine_name" {
  description = "The DB engine name of the DDS instance"
  type        = string
  default     = "DDS-Community"
}

variable "flavor_vcpus" {
  description = "The VCPUs of the flavor"
  type        = number
  default     = 2
}

variable "flavor_memory" {
  description = "The memory of the flavor"
  type        = number
  default     = 4
}

variable "node_type" {
  description = "The type of the DDS instance node"
  type        = string
  default     = "replica"
}

variable "instance_name" {
  description = "The name of the DDS instance"
  type        = string
}

variable "instance_mode" {
  description = "The mode of the DDS instance"
  type        = string
  default     = "ReplicaSet"
}

variable "database_version" {
  description = "The database version of the DDS instance"
  type        = string
  default     = "4.0"
}

variable "storage_engine" {
  description = "The storage engine of the DDS instance"
  type        = string
  default     = "wiredTiger"
}

variable "node_number" {
  description = "The number of nodes of the DDS instance"
  type        = number
  default     = 3
}

variable "node_storage_type" {
  description = "The storage type of the DDS instance node"
  type        = string
  default     = "ULTRAHIGH"
}

variable "node_size" {
  description = "The disk size of the node of the DDS instance"
  type        = number
  default     = 10
}

variable "node_list" {
  description = "The node IDs to be deleted of the DDS instance"
  type        = list(string)
  default     = null
}

variable "instance_port" {
  description = "The database access port of the DDS instance"
  type        = number
  default     = 8635
}

variable "instance_description" {
  description = "The description of the DDS instance"
  type        = string
  default     = ""
}

variable "instance_password" {
  description = "The database access password of the DDS instance"
  sensitive   = true
  type        = string
  default     = ""
}

variable "instance_tags" {
  description = "The tags of the DDS instance"
  type        = map(string)
  default     = {}
}

variable "charging_mode" {
  description = "The charging mode of the DDS instance"
  type        = string
  default     = "postPaid"
}

variable "period_unit" {
  description = "The period unit of the DDS instance"
  type        = string
  default     = null
}

variable "period" {
  description = "The period of the DDS instance"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "The auto renew of the DDS instance"
  type        = string
  default     = "false"
}

resource "huaweicloud_dds_instance" "test" {
  name              = var.instance_name
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id
  mode              = var.instance_mode

  datastore {
    type           = var.engine_name
    version        = var.database_version
    storage_engine = var.storage_engine
  }

  flavor {
    type      = var.node_type
    num       = var.node_number
    spec_code = var.node_spec_code == "" ? try(data.huaweicloud_dds_flavors.test[0].flavors[1].spec_code, null) : var.node_spec_code
    storage   = var.node_storage_type
    size      = var.node_size
    node_list = var.node_list
  }

  port          = var.instance_port
  description   = var.instance_description
  password      = var.instance_password
  tags          = var.instance_tags
  charging_mode = var.charging_mode
  period_unit   = var.period_unit
  period        = var.period
  auto_renew    = var.auto_renew
}
```

**参数说明**：
- **name**：通过引用输入变量 instance_name 进行赋值。
- **availability_zone**：通过引用输入变量 availability_zone 进行赋值，当为空时，自动选择第一个可用分区。
- **vpc_id**：通过引用 huaweicloud_vpc.test.id 进行赋值。
- **subnet_id**：通过引用 huaweicloud_vpc_subnet.test.id 进行赋值。
- **security_group_id**：通过引用 huaweicloud_networking_secgroup.test.id 进行赋值。
- **mode**：通过引用输入变量 instance_mode 进行赋值。
- **datastore**：
  - **type**：通过引用输入变量 engine_name 进行赋值。
  - **version**：通过引用输入变量 database_version 进行赋值。
  - **storage_engine**：通过引用输入变量 storage_engine 进行赋值。
- **flavor**：
  - **type**：通过引用输入变量 node_type 进行赋值。
  - **num**：通过引用输入变量 node_number 进行赋值。
  - **spec_code**：通过引用输入变量 node_spec_code 进行赋值，当为空时，自动选择查询到的规格。
  - **storage**：通过引用输入变量 node_storage_type 进行赋值。
  - **size**：通过引用输入变量 node_size 进行赋值。
  - **node_list**：通过引用输入变量 node_list 进行赋值。
- **port**：通过引用输入变量 instance_port 进行赋值。
- **description**：通过引用输入变量 instance_description 进行赋值。
- **password**：通过引用输入变量 instance_password 进行赋值。
- **tags**：通过引用输入变量 instance_tags 进行赋值。
- **charging_mode**：通过引用输入变量 charging_mode 进行赋值。
- **period_unit**：通过引用输入变量 period_unit 进行赋值。
- **period**：通过引用输入变量 period 进行赋值。
- **auto_renew**：通过引用输入变量 auto_renew 进行赋值。

### 8. 关联DDS实例与LTS

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下关联DDS实例与LTS
variable "log_type" {
  description = "The log type"
  type        = string
  default     = "audit_log"
}

resource "huaweicloud_dds_lts_log" "test" {
  instance_id   = huaweicloud_dds_instance.test.id
  lts_group_id  = huaweicloud_lts_group.test.id
  lts_stream_id = huaweicloud_lts_stream.test.id
  log_type      = var.log_type
}
```

**参数说明**：
- **instance_id**：通过引用 huaweicloud_dds_instance.test.id 进行赋值。
- **lts_group_id**：通过引用 huaweicloud_lts_group.test.id 进行赋值。
- **lts_stream_id**：通过引用 huaweicloud_lts_stream.test.id 进行赋值。
- **log_type**：通过引用输入变量 log_type 进行赋值。

### 9. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 根据脚本变量填写；敏感信息使用占位符
vpc_name            = "your_vpc_name"
subnet_name         = "your_subnet_name"
security_group_name = "your_security_group_name"
group_name          = "your_log_group_name"
stream_name         = "your_log_stream_name"
instance_name       = "your_instance_name"
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
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DDS实例并关联LTS
4. 运行 `terraform show` 查看已创建的DDS实例和LTS关联关系

## 参考信息

- [华为云文档数据库服务产品文档](https://support.huaweicloud.com/dds/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DDS实例关联LTS最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dds/dds-associate-lts)
