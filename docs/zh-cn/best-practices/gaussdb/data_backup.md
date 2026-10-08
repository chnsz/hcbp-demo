# 部署数据备份

## 应用场景

GaussDB是华为云提供的高性能、高可用、高安全的企业级分布式关系型数据库服务，支持集中式和分布式两种部署形态。在实际业务中，为防止误操作、软件故障或数据损坏导致数据丢失，需要定期为数据库实例创建备份，以便在需要时将数据恢复到指定备份点。

本最佳实践将介绍如何使用Terraform自动化部署GaussDB实例并为其创建手动备份，包括VPC、子网、安全组等网络资源创建，GaussDB实例配置以及手动备份创建。

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
- [GaussDB备份（huaweicloud_gaussdb_backup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_backup)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones.test
    └── huaweicloud_gaussdb_instance.test

huaweicloud_vpc.test
    ├── huaweicloud_vpc_subnet.test
    │   └── huaweicloud_gaussdb_instance.test
    └── huaweicloud_networking_secgroup_rule.test

huaweicloud_networking_secgroup.test
    ├── huaweicloud_networking_secgroup_rule.test
    └── huaweicloud_gaussdb_instance.test

random_password.test
    └── huaweicloud_gaussdb_instance.test

huaweicloud_gaussdb_instance.test
    └── huaweicloud_gaussdb_backup.test
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
  nullable    = false
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  nullable    = false
  default     = "172.16.0.0/16"
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：通过引用输入变量 vpc_name 进行赋值
- **cidr**：通过引用输入变量 vpc_cidr 进行赋值

### 3. 创建虚拟私有云子网

在TF文件（如main.tf）中添加以下脚本以创建虚拟私有云子网：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云子网资源
variable "subnet_name" {
  description = "The subnet name"
  type        = string
  nullable    = false
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  nullable    = false
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  nullable    = false
  default     = ""
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**参数说明**：
- **vpc_id**：通过引用虚拟私有云资源的ID进行赋值
- **name**：通过引用输入变量 subnet_name 进行赋值
- **cidr**：通过引用输入变量 subnet_cidr 进行赋值，若为空则根据VPC的CIDR自动计算
- **gateway_ip**：通过引用输入变量 subnet_gateway_ip 进行赋值，若为空则根据子网CIDR自动计算

### 4. 创建安全组

在TF文件（如main.tf）中添加以下脚本以创建安全组：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源
variable "security_group_name" {
  description = "The security group name"
  type        = string
  nullable    = false
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：通过引用输入变量 security_group_name 进行赋值
- **delete_default_rules**：设置为true以删除安全组默认规则，便于后续按需添加规则

### 5. 创建安全组规则

在TF文件（如main.tf）中添加以下脚本以创建安全组规则：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组规则资源
variable "security_group_rule_ports" {
  description = "The security group ingress rule ports"
  type        = string
  nullable    = false
  default     = "2379-2380,5000-5001,5432-5532,6000,6500,12016,20050"
}

resource "huaweicloud_networking_secgroup_rule" "test" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "ingress"
  ethertype         = "IPv4"
  remote_ip_prefix  = huaweicloud_vpc.test.cidr
  ports             = var.security_group_rule_ports
  protocol          = "tcp"
}
```

**参数说明**：
- **security_group_id**：通过引用安全组资源的ID进行赋值
- **direction**：设置为ingress表示入方向规则
- **ethertype**：设置为IPv4表示IPv4协议
- **remote_ip_prefix**：通过引用虚拟私有云的CIDR进行赋值
- **ports**：通过引用输入变量 security_group_rule_ports 进行赋值，用于开放GaussDB实例所需的端口
- **protocol**：设置为tcp表示TCP协议

### 6. 查询可用区列表

在TF文件（如main.tf）中添加以下脚本以查询可用区列表：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区列表数据源
variable "instance_availability_zones" {
  description = "The availability zones for the GaussDB instance, separated by commas"
  type        = string
  nullable    = false
  default     = ""
}

data "huaweicloud_availability_zones" "test" {
  count = var.instance_availability_zones == "" ? 1 : 0
}
```

**参数说明**：
- **count**：当输入变量 instance_availability_zones 为空时创建该数据源，用于自动获取可用区列表

### 7. 创建随机密码

在TF文件（如main.tf）中添加以下脚本以创建随机密码：

```hcl
# 创建随机密码资源
resource "random_password" "test" {
  length           = 16
  min_upper        = 1
  min_lower        = 1
  min_numeric      = 1
  min_special      = 1
  special          = true
  override_special = "~!@#%^*-_=+?"
}
```

**参数说明**：
- **length**：密码长度，设置为16
- **min_upper**：至少包含的大写字母数量，设置为1
- **min_lower**：至少包含的小写字母数量，设置为1
- **min_numeric**：至少包含的数字数量，设置为1
- **min_special**：至少包含的特殊字符数量，设置为1
- **special**：设置为true表示允许使用特殊字符
- **override_special**：允许使用的特殊字符集合

### 8. 创建GaussDB实例

在TF文件（如main.tf）中添加以下脚本以创建GaussDB实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建GaussDB实例资源
variable "instance_name" {
  description = "The name of the GaussDB instance"
  type        = string
  nullable    = false
}

variable "instance_flavor" {
  description = "The flavor of the GaussDB instance"
  type        = string
  nullable    = false
  default     = "gaussdb.opengauss.ee.c3.xlarge.x864.ha"
}

variable "instance_password" {
  description = "The password for the GaussDB instance"
  type        = string
  sensitive   = true
  nullable    = false
  default     = ""
}

variable "instance_db_port" {
  description = "The database port of the GaussDB instance"
  type        = number
  nullable    = false
  default     = 5432
}

variable "enterprise_project_id" {
  description = "The enterprise project ID of the GaussDB instance"
  type        = string
  default     = null
}

variable "instance_ha_mode" {
  description = "The HA mode of the GaussDB instance"
  type        = string
  nullable    = false
  default     = "centralization_standard"
}

variable "instance_ha_replication_mode" {
  description = "The HA replication mode of the GaussDB instance"
  type        = string
  nullable    = false
  default     = "sync"
}

variable "instance_ha_consistency" {
  description = "The HA consistency of the GaussDB instance"
  type        = string
  nullable    = false
  default     = "strong"
}

variable "instance_volume_type" {
  description = "The storage volume type of the GaussDB instance"
  type        = string
  nullable    = false
  default     = "ULTRAHIGH"
}

variable "instance_volume_size" {
  description = "The storage volume size (GB) of the GaussDB instance"
  type        = number
  nullable    = false
  default     = 40
}

resource "huaweicloud_gaussdb_instance" "test" {
  name                  = var.instance_name
  flavor                = var.instance_flavor
  password              = var.instance_password != "" ? var.instance_password : random_password.test.result
  vpc_id                = huaweicloud_vpc.test.id
  subnet_id             = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  availability_zone     = var.instance_availability_zones != "" ? var.instance_availability_zones : join(",", slice(data.huaweicloud_availability_zones.test[0].names, 0, 3))
  port                  = var.instance_db_port
  enterprise_project_id = var.enterprise_project_id

  ha {
    mode             = var.instance_ha_mode
    replication_mode = var.instance_ha_replication_mode
    consistency      = var.instance_ha_consistency
  }

  replica_num = 3

  volume {
    type = var.instance_volume_type
    size = var.instance_volume_size
  }

  lifecycle {
    ignore_changes = [
      flavor,
    ]
  }
}
```

**参数说明**：
- **name**：通过引用输入变量 instance_name 进行赋值
- **flavor**：通过引用输入变量 instance_flavor 进行赋值
- **password**：通过引用输入变量 instance_password 进行赋值，若为空则使用随机密码资源生成的结果
- **vpc_id**：通过引用虚拟私有云资源的ID进行赋值
- **subnet_id**：通过引用虚拟私有云子网资源的ID进行赋值
- **security_group_id**：通过引用安全组资源的ID进行赋值
- **availability_zone**：通过引用输入变量 instance_availability_zones 进行赋值，若为空则自动使用前3个可用区
- **port**：通过引用输入变量 instance_db_port 进行赋值
- **enterprise_project_id**：通过引用输入变量 enterprise_project_id 进行赋值
- **ha.mode**：通过引用输入变量 instance_ha_mode 进行赋值
- **ha.replication_mode**：通过引用输入变量 instance_ha_replication_mode 进行赋值
- **ha.consistency**：通过引用输入变量 instance_ha_consistency 进行赋值
- **replica_num**：副本数量，设置为3
- **volume.type**：通过引用输入变量 instance_volume_type 进行赋值
- **volume.size**：通过引用输入变量 instance_volume_size 进行赋值
- **lifecycle.ignore_changes**：忽略flavor字段的变更，防止实例规格被意外修改

### 9. 创建GaussDB手动备份

在TF文件（如main.tf）中添加以下脚本以创建GaussDB手动备份：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建GaussDB备份资源
variable "backup_name" {
  description = "The name for the manual backup"
  type        = string
  nullable    = false
}

variable "backup_description" {
  description = "The description for the manual backup"
  type        = string
  nullable    = false
  default     = ""
}

resource "huaweicloud_gaussdb_backup" "test" {
  instance_id = huaweicloud_gaussdb_instance.test.id
  name        = var.backup_name
  description = var.backup_description
}
```

**参数说明**：
- **instance_id**：通过引用GaussDB实例资源的ID进行赋值
- **name**：通过引用输入变量 backup_name 进行赋值
- **description**：通过引用输入变量 backup_description 进行赋值

### 10. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 网络资源变量
vpc_name            = "your_vpc_name"
subnet_name         = "your_subnet_name"
security_group_name = "your_security_group_name"

# GaussDB实例变量
instance_name = "your_gaussdb_instance_name"

# 手动备份变量
backup_name = "your_manual_backup_name"
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

### 11. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建GaussDB实例及其手动备份
4. 运行 `terraform show` 查看已创建的GaussDB实例及其手动备份

## 参考信息

- [华为云GaussDB产品文档](https://support.huaweicloud.com/gaussdb/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [GaussDB数据备份最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/gaussdb/data-backup)
