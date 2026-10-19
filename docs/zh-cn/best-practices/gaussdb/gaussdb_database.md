# 部署GaussDB数据库

## 应用场景

GaussDB是华为云提供的高性能、高可用、高安全的企业级分布式关系型数据库服务，支持集中式和分布式两种部署形态。在实际业务中，用户通常需要在GaussDB OpenGauss实例中创建独立的业务数据库，并为数据库指定字符集、所有者、排序规则等属性，以满足不同应用的兼容性与数据隔离要求。

本最佳实践将介绍如何使用Terraform自动化部署GaussDB数据库，包括VPC创建、子网配置、安全组及安全组规则配置、可用区查询、GaussDB实例创建以及数据库创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [随机密码（random_password）](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [GaussDB实例（huaweicloud_gaussdb_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_instance)
- [GaussDB数据库（huaweicloud_gaussdb_database）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_database)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones.test
    └── huaweicloud_gaussdb_instance.test
        └── huaweicloud_gaussdb_database.test

huaweicloud_vpc.test
    ├── huaweicloud_vpc_subnet.test
    │   └── huaweicloud_gaussdb_instance.test
    └── huaweicloud_networking_secgroup_rule.test

huaweicloud_networking_secgroup.test
    ├── huaweicloud_networking_secgroup_rule.test
    └── huaweicloud_gaussdb_instance.test

random_password.test
    └── huaweicloud_gaussdb_instance.test
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

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = ""
}

resource "huaweicloud_vpc" "test" {
  name                  = var.vpc_name
  cidr                  = var.vpc_cidr
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量 vpc_name 进行赋值
- **cidr**：VPC的CIDR网段，通过引用输入变量 vpc_cidr 进行赋值
- **enterprise_project_id**：企业项目ID，当输入变量 enterprise_project_id 非空时引用其值，否则为null

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
}

variable "gateway_ip" {
  description = "The gateway IP address of the subnet"
  type        = string
  default     = ""
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.gateway_ip
}
```

**参数说明**：
- **vpc_id**：子网所属的VPC ID，引用前一步创建的VPC的ID
- **name**：子网名称，通过引用输入变量 subnet_name 进行赋值
- **cidr**：子网CIDR网段，当输入变量 subnet_cidr 为空时基于VPC网段自动计算，否则引用其值
- **gateway_ip**：子网网关IP，当输入变量 gateway_ip 为空时基于子网网段自动计算，否则引用其值

### 4. 查询可用区列表

在TF文件（如main.tf）中添加以下脚本以查询可用区列表：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区列表数据源
data "huaweicloud_availability_zones" "test" {}
```

### 5. 创建安全组及安全组规则

在TF文件（如main.tf）中添加以下脚本以创建安全组及安全组规则：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组及安全组规则资源
variable "security_group_name" {
  description = "The security group name"
  type        = string
}

variable "security_group_rule_ports" {
  description = "The security group ingress rule ports"
  type        = string
  default     = "2379-2380,5000-5001,5432-5532,6000,6500,12016,20050"
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}

resource "huaweicloud_networking_secgroup_rule" "test" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  ports             = var.security_group_rule_ports
  remote_ip_prefix  = huaweicloud_vpc.test.cidr
}
```

**参数说明**：
- **name**：安全组名称，通过引用输入变量 security_group_name 进行赋值
- **delete_default_rules**：是否删除安全组默认规则，设置为true
- **security_group_id**：安全组规则所属的安全组ID，引用前一步创建的安全组的ID
- **direction**：规则方向，设置为ingress表示入方向
- **ethertype**：IP协议类型，设置为IPv4
- **protocol**：协议类型，设置为tcp
- **ports**：规则端口范围，通过引用输入变量 security_group_rule_ports 进行赋值，需包含GaussDB实例运行所需的系统端口与数据库端口
- **remote_ip_prefix**：远端IP网段，引用VPC的CIDR网段

### 6. 创建随机密码

在TF文件（如main.tf）中添加以下脚本以在未指定实例密码时自动生成随机密码：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建随机密码资源
variable "instance_password" {
  description = "The password for the GaussDB instance"
  type        = string
  default     = ""
  sensitive   = true
}

resource "random_password" "test" {
  count = var.instance_password == "" ? 1 : 0

  length           = 12
  special          = true
  override_special = "!@%^*-_=+"
  min_lower        = 1
  min_upper        = 1
  min_numeric      = 1
  min_special      = 1
}
```

**参数说明**：
- **count**：当输入变量 instance_password 为空时创建1个随机密码，否则不创建
- **length**：密码长度，设置为12
- **special**：是否包含特殊字符，设置为true
- **override_special**：允许使用的特殊字符集合
- **min_lower**、**min_upper**、**min_numeric**、**min_special**：分别指定小写字母、大写字母、数字和特殊字符的最小数量

### 7. 创建GaussDB实例

在TF文件（如main.tf）中添加以下脚本以创建GaussDB实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建GaussDB实例资源
variable "instance_name" {
  description = "The GaussDB instance name"
  type        = string
}

variable "instance_volume_type" {
  description = "The storage volume type"
  type        = string
  default     = "ULTRAHIGH"
}

variable "instance_volume_size" {
  description = "The storage volume size in GB"
  type        = number
  default     = 40
}

resource "huaweicloud_gaussdb_instance" "test" {
  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id
  flavor            = "gaussdb.opengauss.ee.dn.m6.2xlarge.8.in"
  name              = var.instance_name
  password          = var.instance_password != "" ? var.instance_password : try(random_password.test[0].result, null)
  sharding_num      = 1
  coordinator_num   = 2
  replica_num       = 3
  availability_zone = join(",", try(slice(data.huaweicloud_availability_zones.test.names, 0, 3), []))

  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null

  ha {
    mode             = "enterprise"
    replication_mode = "sync"
    consistency      = "strong"
  }

  volume {
    type = var.instance_volume_type
    size = var.instance_volume_size
  }
}
```

**参数说明**：
- **vpc_id**：实例所属的VPC ID，引用前一步创建的VPC的ID
- **subnet_id**：实例所属的子网ID，引用前一步创建的子网的ID
- **security_group_id**：实例所属的安全组ID，引用前一步创建的安全组的ID
- **flavor**：实例规格，设置为 `gaussdb.opengauss.ee.dn.m6.2xlarge.8.in`
- **name**：实例名称，通过引用输入变量 instance_name 进行赋值
- **password**：实例密码，当输入变量 instance_password 非空时引用其值，否则引用随机密码资源生成的结果
- **sharding_num**：分片数量，设置为1
- **coordinator_num**：协调节点数量，设置为2
- **replica_num**：副本数量，设置为3
- **availability_zone**：可用区，从可用区列表数据源中取前三个可用区并以逗号拼接
- **enterprise_project_id**：企业项目ID，当输入变量 enterprise_project_id 非空时引用其值，否则为null
- **ha.mode**：高可用模式，设置为enterprise
- **ha.replication_mode**：复制模式，设置为sync
- **ha.consistency**：一致性级别，设置为strong
- **volume.type**：存储类型，通过引用输入变量 instance_volume_type 进行赋值
- **volume.size**：存储大小，通过引用输入变量 instance_volume_size 进行赋值

### 8. 创建GaussDB数据库

在TF文件（如main.tf）中添加以下脚本以在GaussDB实例中创建数据库：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建GaussDB数据库资源
variable "database_name" {
  description = "The name of the database"
  type        = string
}

variable "character_set" {
  description = "The database character set"
  type        = string
  default     = "UTF8"
}

variable "owner" {
  description = "The database user"
  type        = string
  default     = "root"
}

variable "template" {
  description = "The name of the database template"
  type        = string
  default     = ""
}

variable "lc_collate" {
  description = "The database collation"
  type        = string
  default     = "C"
}

variable "lc_ctype" {
  description = "The database classification"
  type        = string
  default     = "C"
}

resource "huaweicloud_gaussdb_database" "test" {
  instance_id   = huaweicloud_gaussdb_instance.test.id
  name          = var.database_name
  character_set = var.character_set
  owner         = var.owner
  template      = var.template
  lc_collate    = var.lc_collate
  lc_ctype      = var.lc_ctype
}
```

**参数说明**：
- **instance_id**：数据库所属的GaussDB实例ID，引用前一步创建的GaussDB实例的ID
- **name**：数据库名称，通过引用输入变量 database_name 进行赋值，长度1-63个字符，不能以pg或数字开头，且不能为模板数据库名称
- **character_set**：数据库字符集，通过引用输入变量 character_set 进行赋值，默认为UTF8
- **owner**：数据库所有者，通过引用输入变量 owner 进行赋值，默认为root，且不能为系统用户（rdsAdmin、rdsMetric、rdsBackup、rdsRepl）
- **template**：数据库模板名称，通过引用输入变量 template 进行赋值，可设置为template0
- **lc_collate**：数据库排序规则，通过引用输入变量 lc_collate 进行赋值，默认为C
- **lc_ctype**：数据库字符分类，通过引用输入变量 lc_ctype 进行赋值，默认为C

### 9. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 资源变量
vpc_name            = "tf_test_vpc"
subnet_name         = "tf_test_subnet"
security_group_name = "tf_test_security_group"
instance_name       = "tf_test_gaussdb_instance"
database_name       = "tf_test_db"
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
3. 确认资源计划无误后，运行 `terraform apply` 开始创建GaussDB数据库
4. 运行 `terraform show` 查看已创建的GaussDB数据库

## 参考信息

- [华为云GaussDB产品文档](https://support.huaweicloud.com/gaussdb/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [GaussDB数据库最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/gaussdb/gaussdb-database)
