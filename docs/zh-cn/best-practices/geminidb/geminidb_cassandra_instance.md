# 部署GeminiDB Cassandra实例

## 应用场景

GeminiDB Cassandra是华为云提供的兼容Apache Cassandra协议的分布式NoSQL数据库服务，具备高可用、高可靠、弹性扩展等特性，适用于海量数据存储、高并发读写以及宽表模型等业务场景。

本最佳实践将介绍如何使用Terraform自动化部署一个GeminiDB Cassandra实例，包括VPC、子网、安全组的创建，实例规格的自动查询，实例密码的自动生成，以及备份策略与实例备份的配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [GeminiDB NoSQL规格列表（data.huaweicloud_gaussdb_nosql_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/gaussdb_nosql_flavors)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [随机密码（random_password）](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [GeminiDB实例（huaweicloud_geminidb_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/geminidb_instance)
- [GeminiDB备份（huaweicloud_geminidb_backup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/geminidb_backup)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones.test
    ├── data.huaweicloud_gaussdb_nosql_flavors.test
    └── huaweicloud_geminidb_instance.test

huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test
            └── huaweicloud_geminidb_instance.test

huaweicloud_networking_secgroup.test
    └── huaweicloud_geminidb_instance.test

random_password.test
    └── huaweicloud_geminidb_instance.test

huaweicloud_geminidb_instance.test
    └── huaweicloud_geminidb_backup.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建VPC和子网

在TF文件（如main.tf）中添加以下脚本以创建VPC和子网：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC和子网
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
}

variable "gateway_ip" {
  description = "The gateway IP address of the subnet"
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
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.gateway_ip
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量 vpc_name 进行赋值
- **cidr**：VPC的网段，通过引用输入变量 vpc_cidr 进行赋值
- **vpc_id**：子网所属的VPC ID，引用VPC资源的ID进行赋值
- **cidr**：子网的网段，当输入变量 subnet_cidr 为空时，通过 cidrsubnet 函数基于VPC网段自动计算
- **gateway_ip**：子网的网关IP，当输入变量 gateway_ip 为空时，通过 cidrhost 函数基于子网网段自动计算

### 3. 查询可用区和GeminiDB NoSQL规格

在TF文件（如main.tf）中添加以下脚本以查询可用区和GeminiDB NoSQL规格：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区和GeminiDB NoSQL规格
variable "vcpus" {
  description = "The number of vCPUs"
  type        = string
  default     = "2"
}

variable "availability_zone" {
  description = "The availability zone to which the GeminiDB Cassandra instance belongs"
  type        = string
  default     = ""
}

data "huaweicloud_availability_zones" "test" {
}

data "huaweicloud_gaussdb_nosql_flavors" "test" {
  vcpus             = var.vcpus
  engine            = "cassandra"
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
}
```

**参数说明**：
- **vcpus**：规格的vCPU数量，通过引用输入变量 vcpus 进行赋值
- **engine**：数据库引擎类型，固定为 cassandra
- **availability_zone**：规格所属的可用区，当输入变量 availability_zone 为空时，自动使用可用区列表中的第一个可用区

### 4. 创建安全组

在TF文件（如main.tf）中添加以下脚本以创建安全组：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组
variable "security_group_name" {
  description = "The security group name"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组名称，通过引用输入变量 security_group_name 进行赋值
- **delete_default_rules**：是否删除安全组默认规则，设置为 true 以便按需自定义访问规则

### 5. 生成随机密码

在TF文件（如main.tf）中添加以下脚本以在未提供密码时自动生成随机密码：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下生成随机密码
variable "instance_password" {
  description = "The password for the GeminiDB Cassandra instance"
  type        = string
  default     = ""
  sensitive   = true
}

resource "random_password" "test" {
  count            = var.instance_password == "" ? 1 : 0

  length           = 12
  special          = true
  override_special = "!@%^*-_=+"
  min_upper        = 1
  min_lower        = 1
  min_numeric      = 1
  min_special      = 1
}
```

**参数说明**：
- **count**：当输入变量 instance_password 为空时创建随机密码，否则不创建
- **length**：密码长度，设置为 12
- **special**：是否包含特殊字符，设置为 true
- **override_special**：允许使用的特殊字符集合
- **min_upper**、**min_lower**、**min_numeric**、**min_special**：分别指定大写字母、小写字母、数字和特殊字符的最小数量

### 6. 创建GeminiDB Cassandra实例

在TF文件（如main.tf）中添加以下脚本以创建GeminiDB Cassandra实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建GeminiDB Cassandra实例
variable "instance_name" {
  description = "The GeminiDB Cassandra instance name"
  type        = string
}

variable "instance_mode" {
  description = "The instance mode. Valid values are Cluster, Single"
  type        = string
  default     = "Cluster"
}

variable "instance_db_port" {
  description = "The Cassandra database port"
  type        = number
  default     = 9042
}

variable "instance_ssl_option" {
  description = "The SSL option. Valid values are on, off"
  type        = string
  default     = "on"
}

variable "instance_flavor_num" {
  description = "The number of nodes in the Cassandra cluster"
  type        = number
  default     = 3
}

variable "instance_flavor_size" {
  description = "The storage size in GB per node"
  type        = number
  default     = 100
}

variable "instance_flavor_storage" {
  description = "The storage type. Valid values are ULTRAHIGH, ESSD"
  type        = string
  default     = "ULTRAHIGH"
}

variable "instance_flavor_spec_code" {
  description = "The resource specification code. If empty, it will be queried from flavors data source"
  type        = string
  default     = ""
}

variable "instance_backup_time_window" {
  description = "The backup time window in HH:MM-HH:MM format"
  type        = string
}

variable "instance_backup_keep_days" {
  description = "The number of days to retain backups"
  type        = number
}

variable "tags" {
  description = "The key/value pairs to associate with the GeminiDB Cassandra instance"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_geminidb_instance" "test" {
  name              = var.instance_name
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id
  password          = var.instance_password != "" ? var.instance_password : try(random_password.test[0].result)
  mode              = var.instance_mode
  port              = var.instance_db_port
  ssl_option        = var.instance_ssl_option

  datastore {
    type           = "cassandra"
    version        = "3.11"
    storage_engine = "rocksDB"
  }

  flavor {
    num       = var.instance_flavor_num
    size      = var.instance_flavor_size
    storage   = var.instance_flavor_storage
    spec_code = var.instance_flavor_spec_code != "" ? var.instance_flavor_spec_code : try(data.huaweicloud_gaussdb_nosql_flavors.test.flavors[0].name, null)
  }

  backup_strategy {
    start_time = var.instance_backup_time_window
    keep_days  = var.instance_backup_keep_days
  }

  charging_mode = "prePaid"
  period_unit   = "month"
  auto_renew    = "true"
  period        = 1

  tags = var.tags

  lifecycle {
    ignore_changes = [
      flavor.0.spec_code,
    ]
  }
}
```

**参数说明**：
- **name**：实例名称，通过引用输入变量 instance_name 进行赋值
- **availability_zone**：实例所属的可用区，当输入变量 availability_zone 为空时，自动使用可用区列表中的第一个可用区
- **vpc_id**：实例所属的VPC ID，引用VPC资源的ID进行赋值
- **subnet_id**：实例所属的子网 ID，引用子网资源的ID进行赋值
- **security_group_id**：实例所属的安全组ID，引用安全组资源的ID进行赋值
- **password**：实例密码，当输入变量 instance_password 不为空时使用该值，否则使用自动生成的随机密码
- **mode**：实例模式，通过引用输入变量 instance_mode 进行赋值
- **port**：数据库端口，通过引用输入变量 instance_db_port 进行赋值
- **ssl_option**：SSL选项，通过引用输入变量 instance_ssl_option 进行赋值
- **datastore**：数据库引擎信息，type 为 cassandra，version 为 3.11，storage_engine 为 rocksDB
- **flavor**：实例规格信息，num、size、storage 分别通过引用输入变量 instance_flavor_num、instance_flavor_size、instance_flavor_storage 进行赋值，spec_code 在输入变量 instance_flavor_spec_code 为空时自动从规格数据源中获取
- **backup_strategy**：备份策略，start_time 和 keep_days 分别通过引用输入变量 instance_backup_time_window 和 instance_backup_keep_days 进行赋值
- **charging_mode**、**period_unit**、**auto_renew**、**period**：计费模式相关参数，分别设置为 prePaid、month、true、1
- **tags**：实例标签，通过引用输入变量 tags 进行赋值
- **lifecycle**：生命周期配置，忽略 flavor 中 spec_code 的变化

### 7. 创建GeminiDB备份

在TF文件（如main.tf）中添加以下脚本以创建GeminiDB备份：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建GeminiDB备份
variable "backup_name" {
  description = "The name for instance backups"
  type        = string
}

variable "backup_description" {
  description = "The description for instance backups"
  type        = string
  default     = "Terraform created backup"
}

resource "huaweicloud_geminidb_backup" "test" {
  instance_id = huaweicloud_geminidb_instance.test.id
  name        = var.backup_name
  description = var.backup_description

  depends_on = [huaweicloud_geminidb_instance.test]
}
```

**参数说明**：
- **instance_id**：备份所属的实例ID，引用GeminiDB实例资源的ID进行赋值
- **name**：备份名称，通过引用输入变量 backup_name 进行赋值
- **description**：备份描述，通过引用输入变量 backup_description 进行赋值
- **depends_on**：显式声明依赖关系，确保备份在实例创建完成后创建

### 8. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 资源变量
vpc_name                    = "tf_test_vpc"
subnet_name                 = "tf_test_subnet"
security_group_name         = "tf_test_security_group"
instance_name               = "tf_test_geminidb_cassandra"
instance_backup_time_window = "03:00-04:00"
instance_backup_keep_days   = 14
backup_name                 = "tf_test_backup"
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
3. 确认资源计划无误后，运行 `terraform apply` 开始创建GeminiDB Cassandra实例
4. 运行 `terraform show` 查看已创建的GeminiDB Cassandra实例

## 参考信息

- [华为云GeminiDB产品文档](https://support.huaweicloud.com/geminidb/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [GeminiDB Cassandra实例最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/geminidb/geminidb-cassandra-instance)
