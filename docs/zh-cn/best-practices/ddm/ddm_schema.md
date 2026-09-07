# 部署逻辑库

## 应用场景

分布式数据库中间件（DDM）通过分库分表解决传统数据库的容量和性能瓶颈，实现海量数据的高并发访问。在业务数据量持续增长、单库容量或性能达到上限时，您需要为DDM实例创建逻辑库，将数据分散到多个数据节点中。

本最佳实践将介绍如何使用Terraform完成DDM逻辑库的部署，包括创建VPC、子网、安全组，部署RDS数据节点与DDM实例，并创建关联数据节点的逻辑库。通过本实践，您可以快速掌握使用Terraform自动化部署DDM逻辑库的方法，为后续的数据分片管理奠定基础。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用分区（huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [RDS规格（huaweicloud_rds_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/rds_flavors)
- [DDM引擎（huaweicloud_ddm_engines）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/ddm_engines)
- [DDM规格（huaweicloud_ddm_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/ddm_flavors)

### 资源

- [虚拟私有云VPC（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [随机密码（random_password）](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [RDS实例（huaweicloud_rds_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rds_instance)
- [DDM实例（huaweicloud_ddm_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ddm_instance)
- [DDM逻辑库（huaweicloud_ddm_schema）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ddm_schema)

### 资源/数据源依赖关系

```
huaweicloud_availability_zones
    └── huaweicloud_rds_instance
    └── huaweicloud_ddm_instance
huaweicloud_rds_flavors
    └── huaweicloud_rds_instance
huaweicloud_ddm_engines
    └── huaweicloud_ddm_flavors
    └── huaweicloud_ddm_instance
huaweicloud_ddm_flavors
    └── huaweicloud_ddm_instance
huaweicloud_vpc
    └── huaweicloud_vpc_subnet
    └── huaweicloud_rds_instance
    └── huaweicloud_ddm_instance
huaweicloud_vpc_subnet
    └── huaweicloud_rds_instance
    └── huaweicloud_ddm_instance
huaweicloud_networking_secgroup
    └── huaweicloud_rds_instance
    └── huaweicloud_ddm_instance
random_password
    └── huaweicloud_rds_instance
    └── huaweicloud_ddm_schema
huaweicloud_rds_instance
    └── huaweicloud_ddm_schema
huaweicloud_ddm_instance
    └── huaweicloud_ddm_schema
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用分区

在TF文件（如main.tf）中添加以下脚本，查询DDM实例和RDS实例可用的可用分区：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用分区
variable "availability_zones" {
  description = "DDM实例所属的可用分区"
  type        = list(string)
  default     = []
  nullable    = false
}

data "huaweicloud_availability_zones" "test" {
  count = length(var.availability_zones) == 0 ? 1 : 0
}
```

**参数说明**：
- **availability_zones**：通过引用输入变量 availability_zones 进行赋值，当未指定可用分区时，将自动查询可用的可用分区。

### 3. 创建VPC

在TF文件（如main.tf）中添加以下脚本，创建VPC：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC
variable "vpc_name" {
  description = "VPC名称"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC的CIDR地址块"
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

### 4. 创建子网

在TF文件（如main.tf）中添加以下脚本，创建子网：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建子网
variable "subnet_name" {
  description = "子网名称"
  type        = string
}

variable "subnet_cidr" {
  description = "子网的CIDR地址块"
  type        = string
  default     = ""
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "子网的网关IP"
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
- **cidr**：通过引用输入变量 subnet_cidr 进行赋值，当未指定时，将基于VPC的CIDR自动计算。
- **gateway_ip**：通过引用输入变量 subnet_gateway_ip 进行赋值，当未指定时，将自动计算。

### 5. 创建安全组

在TF文件（如main.tf）中添加以下脚本，创建安全组：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组
variable "security_group_name" {
  description = "安全组名称"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name = var.security_group_name
}
```

**参数说明**：
- **name**：通过引用输入变量 security_group_name 进行赋值。

### 6. 生成随机密码

在TF文件（如main.tf）中添加以下脚本，当未指定RDS实例密码时，自动生成随机密码：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下生成随机密码
variable "rds_instance_password" {
  description = "RDS实例的密码"
  type        = string
  sensitive   = true
  default     = ""
  nullable    = false
}

resource "random_password" "test" {
  count = var.rds_instance_password == "" ? 1 : 0

  length           = 12
  special          = true
  override_special = "~!@#%^*-_+?"
  min_upper        = 1
  min_lower        = 1
  min_numeric      = 1
  min_special      = 1
}
```

**参数说明**：
- **count**：当未指定RDS实例密码时，创建随机密码资源。
- **length**：密码长度。
- **special**：是否包含特殊字符。
- **override_special**：特殊字符集合。
- **min_upper**：至少包含的大写字母数量。
- **min_lower**：至少包含的小写字母数量。
- **min_numeric**：至少包含的数字数量。
- **min_special**：至少包含的特殊字符数量。

### 7. 查询RDS规格

在TF文件（如main.tf）中添加以下脚本，当未指定RDS实例规格时，查询可用的RDS规格：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询RDS规格
variable "instance_flavor" {
  description = "RDS实例的规格"
  type        = string
  default     = ""
  nullable    = false
}

variable "database_type" {
  description = "RDS实例的数据库类型"
  type        = string
  default     = "MySQL"
}

variable "database_version" {
  description = "RDS实例的数据库版本"
  type        = string
  default     = "5.7"
}

variable "instance_mode" {
  description = "RDS实例的部署模式"
  type        = string
  default     = "single"
}

variable "instance_group_type" {
  description = "RDS实例的性能规格"
  type        = string
  default     = "dedicated"
}

variable "instance_flavor_vcpus" {
  description = "RDS实例规格的vCPU数量"
  type        = number
  default     = 2
}

data "huaweicloud_rds_flavors" "test" {
  count = var.instance_flavor == "" ? 1 : 0

  db_type       = var.database_type
  db_version    = var.database_version
  instance_mode = var.instance_mode
  group_type    = var.instance_group_type
  vcpus         = var.instance_flavor_vcpus
}
```

**参数说明**：
- **db_type**：通过引用输入变量 database_type 进行赋值。
- **db_version**：通过引用输入变量 database_version 进行赋值。
- **instance_mode**：通过引用输入变量 instance_mode 进行赋值。
- **group_type**：通过引用输入变量 instance_group_type 进行赋值。
- **vcpus**：通过引用输入变量 instance_flavor_vcpus 进行赋值。

### 8. 创建RDS实例

在TF文件（如main.tf）中添加以下脚本，创建RDS实例作为DDM逻辑库的数据节点：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建RDS实例
variable "rds_instance_name" {
  description = "RDS实例名称"
  type        = string
}

variable "database_port" {
  description = "RDS实例的数据库端口"
  type        = number
  default     = 3306
}

variable "volume_type" {
  description = "RDS实例的磁盘类型"
  type        = string
  default     = "CLOUDSSD"
}

variable "volume_size" {
  description = "RDS实例的磁盘大小"
  type        = number
  default     = 40
}

resource "huaweicloud_rds_instance" "test" {
  name              = var.rds_instance_name
  availability_zone = length(var.availability_zones) == 0 ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1)) : var.availability_zones
  flavor            = var.instance_flavor == "" ? try(data.huaweicloud_rds_flavors.test[0].flavors[0].name, null) : var.instance_flavor
  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id

  db {
    type     = var.database_type
    version  = var.database_version
    port     = var.database_port
    password = var.rds_instance_password == "" ? try(random_password.test[0].result, null) : var.rds_instance_password
  }

  volume {
    type = var.volume_type
    size = var.volume_size
  }
}
```

**参数说明**：
- **name**：通过引用输入变量 rds_instance_name 进行赋值。
- **availability_zone**：通过引用输入变量 availability_zones 或 data.huaweicloud_availability_zones.test 进行赋值。
- **flavor**：通过引用输入变量 instance_flavor 或 data.huaweicloud_rds_flavors.test 进行赋值。
- **vpc_id**：通过引用 huaweicloud_vpc.test.id 进行赋值。
- **subnet_id**：通过引用 huaweicloud_vpc_subnet.test.id 进行赋值。
- **security_group_id**：通过引用 huaweicloud_networking_secgroup.test.id 进行赋值。
- **db.type**：通过引用输入变量 database_type 进行赋值。
- **db.version**：通过引用输入变量 database_version 进行赋值。
- **db.port**：通过引用输入变量 database_port 进行赋值。
- **db.password**：通过引用输入变量 rds_instance_password 或 random_password.test 进行赋值。
- **volume.type**：通过引用输入变量 volume_type 进行赋值。
- **volume.size**：通过引用输入变量 volume_size 进行赋值。

### 9. 查询DDM引擎和规格

在TF文件（如main.tf）中添加以下脚本，当未指定DDM实例的引擎和规格时，查询可用的DDM引擎和规格：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询DDM引擎和规格
variable "instance_engine_id" {
  description = "DDM实例的引擎ID"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_flavor_id" {
  description = "DDM实例的规格ID"
  type        = string
  default     = ""
  nullable    = false
}

data "huaweicloud_ddm_engines" "test" {
  count = var.instance_engine_id == "" ? 1 : 0
}

data "huaweicloud_ddm_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  engine_id = var.instance_engine_id == "" ? try(data.huaweicloud_ddm_engines.test[0].engines[0].id, null)  : var.instance_engine_id
}
```

**参数说明**：
- **engine_id**：通过引用输入变量 instance_engine_id 或 data.huaweicloud_ddm_engines.test 进行赋值。

### 10. 创建DDM实例

在TF文件（如main.tf）中添加以下脚本，创建DDM实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DDM实例
variable "ddm_instance_name" {
  description = "DDM实例名称"
  type        = string
}

variable "instance_node_num" {
  description = "DDM实例的节点数量"
  type        = number
  default     = 2
}

variable "instance_parameters" {
  description = "DDM实例的参数"

  type = list(object({
    name  = string
    value = string
  }))

  default = []
}

resource "huaweicloud_ddm_instance" "test" {
  name               = var.ddm_instance_name
  availability_zones = length(var.availability_zones) == 0 ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1)) : var.availability_zones
  engine_id          = var.instance_engine_id == "" ? try(data.huaweicloud_ddm_engines.test[0].engines[0].id, null)  : var.instance_engine_id
  flavor_id          = var.instance_flavor_id == "" ? try(data.huaweicloud_ddm_flavors.test[0].flavors[0].id, null)  : var.instance_flavor_id
  vpc_id             = huaweicloud_vpc.test.id
  subnet_id          = huaweicloud_vpc_subnet.test.id
  security_group_id  = huaweicloud_networking_secgroup.test.id
  node_num           = var.instance_node_num

  dynamic "parameters" {
    for_each = var.instance_parameters

    content {
      name  = parameters.value.name
      value = parameters.value.value
    }
  }
}
```

**参数说明**：
- **name**：通过引用输入变量 ddm_instance_name 进行赋值。
- **availability_zones**：通过引用输入变量 availability_zones 或 data.huaweicloud_availability_zones.test 进行赋值。
- **engine_id**：通过引用输入变量 instance_engine_id 或 data.huaweicloud_ddm_engines.test 进行赋值。
- **flavor_id**：通过引用输入变量 instance_flavor_id 或 data.huaweicloud_ddm_flavors.test 进行赋值。
- **vpc_id**：通过引用 huaweicloud_vpc.test.id 进行赋值。
- **subnet_id**：通过引用 huaweicloud_vpc_subnet.test.id 进行赋值。
- **security_group_id**：通过引用 huaweicloud_networking_secgroup.test.id 进行赋值。
- **node_num**：通过引用输入变量 instance_node_num 进行赋值。
- **parameters**：通过引用输入变量 instance_parameters 进行赋值，可配置多个参数。

### 11. 创建DDM逻辑库

在TF文件（如main.tf）中添加以下脚本，创建DDM逻辑库并关联数据节点：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DDM逻辑库
variable "schema_name" {
  description = "DDM逻辑库名称"
  type        = string
}

variable "schema_shard_mode" {
  description = "DDM逻辑库的分片模式"
  type        = string
  default     = "single"
}

variable "schema_shard_number" {
  description = "DDM逻辑库的分片数量"
  type        = number
  default     = 1
}

resource "huaweicloud_ddm_schema" "test" {
  instance_id  = huaweicloud_ddm_instance.test.id
  name         = var.schema_name
  shard_mode   = var.schema_shard_mode
  shard_number = var.schema_shard_number

  data_nodes {
    id             = huaweicloud_rds_instance.test.id
    admin_user     = "root"
    admin_password = var.rds_instance_password == "" ? try(random_password.test[0].result, null) : var.rds_instance_password
  }

  lifecycle {
    ignore_changes = [
      data_nodes,
    ]
  }
}
```

**参数说明**：
- **instance_id**：通过引用 huaweicloud_ddm_instance.test.id 进行赋值。
- **name**：通过引用输入变量 schema_name 进行赋值。
- **shard_mode**：通过引用输入变量 schema_shard_mode 进行赋值。
- **shard_number**：通过引用输入变量 schema_shard_number 进行赋值。
- **data_nodes.id**：通过引用 huaweicloud_rds_instance.test.id 进行赋值。
- **data_nodes.admin_user**：固定为root。
- **data_nodes.admin_password**：通过引用输入变量 rds_instance_password 或 random_password.test 进行赋值。
- **lifecycle.ignore_changes**：忽略data_nodes的变更，避免因数据节点变化导致逻辑库被重建。

### 12. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 根据脚本变量填写；敏感信息使用占位符
vpc_name            = "example-vpc"
subnet_name         = "example-subnet"
security_group_name = "example-security-group"
rds_instance_name   = "example-rds-instance"
ddm_instance_name   = "example-ddm-instance"
schema_name         = "example-schema"
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

### 13. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DDM逻辑库
4. 运行 `terraform show` 查看已创建的DDM逻辑库

## 参考信息

- [华为云DDM产品文档](https://support.huaweicloud.com/ddm/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DDM逻辑库最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ddm/ddm-schema)
