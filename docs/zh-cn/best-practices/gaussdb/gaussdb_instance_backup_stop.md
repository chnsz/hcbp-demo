# 部署备份停止

## 应用场景

GaussDB是华为云提供的高性能、高可用、高安全的企业级分布式关系型数据库服务，支持集中式和分布式两种部署形态。在数据库日常运维中，备份任务可能因数据量较大或业务高峰期而长时间运行，占用实例资源并影响业务性能，此时需要及时停止正在进行的备份任务。

本最佳实践将介绍如何使用Terraform自动化部署GaussDB实例并停止其正在进行的备份任务，包括VPC创建、子网配置、安全组配置、GaussDB实例创建以及备份停止操作。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [GaussDB规格列表（data.huaweicloud_gaussdb_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/gaussdb_flavors)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [随机密码（random_password）](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [GaussDB实例（huaweicloud_gaussdb_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_instance)
- [GaussDB备份停止（huaweicloud_gaussdb_backup_stop）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_backup_stop)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones.test
data.huaweicloud_gaussdb_flavors.test
    └── huaweicloud_gaussdb_instance.test
        └── huaweicloud_gaussdb_backup_stop.test

huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test
        └── huaweicloud_gaussdb_instance.test

huaweicloud_networking_secgroup.test
    └── huaweicloud_networking_secgroup_rule.test
        └── huaweicloud_gaussdb_instance.test

random_password.test
    └── huaweicloud_gaussdb_instance.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建虚拟私有云

在TF文件（如main.tf）中添加以下脚本以创建VPC：

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
- **name**：通过引用输入变量 vpc_name 进行赋值
- **cidr**：通过引用输入变量 vpc_cidr 进行赋值，默认值为`192.168.0.0/16`
- **enterprise_project_id**：通过引用输入变量 enterprise_project_id 进行赋值，为空时不指定企业项目

### 3. 创建子网

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
- **vpc_id**：通过引用 huaweicloud_vpc.test.id 进行赋值
- **name**：通过引用输入变量 subnet_name 进行赋值
- **cidr**：通过引用输入变量 subnet_cidr 进行赋值，为空时基于VPC网段自动计算
- **gateway_ip**：通过引用输入变量 gateway_ip 进行赋值，为空时基于VPC网段自动计算

### 4. 查询可用区和GaussDB规格

在TF文件（如main.tf）中添加以下脚本以查询可用区和GaussDB规格：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区列表和GaussDB规格列表
data "huaweicloud_availability_zones" "test" {}

data "huaweicloud_gaussdb_flavors" "test" {
  version = "8.201"
  ha_mode = "centralization_standard"
}
```

**参数说明**：
- **version**：GaussDB版本号，固定为`8.201`
- **ha_mode**：高可用模式，固定为`centralization_standard`

### 5. 创建安全组及规则

在TF文件（如main.tf）中添加以下脚本以创建安全组及规则：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组及安全组规则资源
variable "security_group_name" {
  description = "The security group name"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}

resource "huaweicloud_networking_secgroup_rule" "test" {
  for_each = toset(["ingress", "egress"])

  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = each.key
  ethertype         = "IPv4"
  protocol          = "tcp"
  remote_ip_prefix  = "0.0.0.0/0"
}
```

**参数说明**：
- **name**：通过引用输入变量 security_group_name 进行赋值
- **delete_default_rules**：删除安全组默认规则，固定为`true`
- **direction**：安全组规则方向，分别创建入方向和出方向规则
- **ethertype**：IP协议类型，固定为`IPv4`
- **protocol**：协议类型，固定为`tcp`
- **remote_ip_prefix**：远端IP地址范围，固定为`0.0.0.0/0`

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
- **count**：当输入变量 instance_password 为空时创建随机密码
- **length**：密码长度，固定为`12`
- **special**：是否包含特殊字符，固定为`true`
- **override_special**：允许使用的特殊字符集合
- **min_lower**、**min_upper**、**min_numeric**、**min_special**：各类字符的最小数量

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
  depends_on = [
    huaweicloud_networking_secgroup_rule.test
  ]

  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id
  flavor            = try(data.huaweicloud_gaussdb_flavors.test.flavors[0].spec_code, "")
  name              = var.instance_name
  password          = var.instance_password != "" ? var.instance_password : try(random_password.test[0].result, null)
  availability_zone = join(",", try(slice(data.huaweicloud_availability_zones.test.names, 0, 3), []))

  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null

  ha {
    mode             = "centralization_standard"
    replication_mode = "sync"
    consistency      = "eventual"
    instance_mode    = "basic"
  }

  volume {
    type = var.instance_volume_type
    size = var.instance_volume_size
  }
}
```

**参数说明**：
- **vpc_id**：通过引用 huaweicloud_vpc.test.id 进行赋值
- **subnet_id**：通过引用 huaweicloud_vpc_subnet.test.id 进行赋值
- **security_group_id**：通过引用 huaweicloud_networking_secgroup.test.id 进行赋值
- **flavor**：通过引用 data.huaweicloud_gaussdb_flavors.test.flavors[0].spec_code 进行赋值
- **name**：通过引用输入变量 instance_name 进行赋值
- **password**：通过引用输入变量 instance_password 或随机密码进行赋值
- **availability_zone**：通过引用可用区列表进行赋值，取前三个可用区
- **enterprise_project_id**：通过引用输入变量 enterprise_project_id 进行赋值，为空时不指定企业项目
- **ha.mode**：高可用模式，固定为`centralization_standard`
- **ha.replication_mode**：复制模式，固定为`sync`
- **ha.consistency**：一致性模式，固定为`eventual`
- **ha.instance_mode**：实例模式，固定为`basic`
- **volume.type**：存储类型，通过引用输入变量 instance_volume_type 进行赋值
- **volume.size**：存储大小，通过引用输入变量 instance_volume_size 进行赋值

### 8. 停止GaussDB实例备份

在TF文件（如main.tf）中添加以下脚本以停止GaussDB实例正在进行的备份：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下停止GaussDB实例备份
resource "huaweicloud_gaussdb_backup_stop" "test" {
  instance_id = huaweicloud_gaussdb_instance.test.id
}
```

**参数说明**：
- **instance_id**：通过引用 huaweicloud_gaussdb_instance.test.id 进行赋值

### 9. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
vpc_name            = "tf_test_vpc"
subnet_name         = "tf_test_subnet"
security_group_name = "tf_test_security_group"
instance_name       = "tf_test_gaussdb_instance"
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
3. 确认资源计划无误后，运行 `terraform apply` 开始创建GaussDB实例并停止其正在进行的备份
4. 运行 `terraform show` 查看已创建的GaussDB实例及备份停止操作

## 参考信息

- [华为云GaussDB产品文档](https://support.huaweicloud.com/gaussdb/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [GaussDB备份停止最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/gaussdb/gaussdb-instance-backup-stop)
