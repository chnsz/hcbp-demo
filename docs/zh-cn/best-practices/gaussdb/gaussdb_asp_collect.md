# 部署ASP采集

## 应用场景

GaussDB是华为云提供的高性能、高可用、高安全的企业级分布式关系型数据库服务，支持集中式和分布式两种部署形态。ASP（Application Support Package）采集用于按指定时间范围收集GaussDB实例的运行快照数据，生成可供下载的分析报告，帮助运维人员定位性能问题、分析慢SQL与等待事件。

本最佳实践将介绍如何使用Terraform自动化部署GaussDB ASP采集，包括VPC创建、子网配置、安全组创建、GaussDB实例配置以及ASP采集任务创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [网络ACL安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [随机密码（random_password）](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [GaussDB实例（huaweicloud_gaussdb_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_instance)
- [GaussDB ASP采集（huaweicloud_gaussdb_asp_collect）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_asp_collect)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones.test
    └── huaweicloud_gaussdb_instance.test
        └── huaweicloud_gaussdb_asp_collect.test

huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test
        └── huaweicloud_gaussdb_instance.test

huaweicloud_networking_secgroup.test
    └── huaweicloud_gaussdb_instance.test

random_password.test
    └── huaweicloud_gaussdb_instance.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建虚拟私有云

在TF文件（如main.tf）中添加以下脚本：

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
- **cidr**：通过引用输入变量 vpc_cidr 进行赋值
- **enterprise_project_id**：通过引用输入变量 enterprise_project_id 进行赋值，为空时使用默认企业项目

### 3. 创建虚拟私有云子网

在TF文件（如main.tf）中添加以下脚本：

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
- **vpc_id**：通过引用虚拟私有云资源的ID进行赋值
- **name**：通过引用输入变量 subnet_name 进行赋值
- **cidr**：通过引用输入变量 subnet_cidr 进行赋值，为空时基于VPC网段自动计算
- **gateway_ip**：通过引用输入变量 gateway_ip 进行赋值，为空时基于子网网段自动计算

### 4. 查询可用区列表

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区列表
data "huaweicloud_availability_zones" "test" {}
```

**参数说明**：
- 该数据源无需输入参数，用于获取当前region下可用的可用区名称列表

### 5. 创建安全组

在TF文件（如main.tf）中添加以下脚本：

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
```

**参数说明**：
- **name**：通过引用输入变量 security_group_name 进行赋值
- **delete_default_rules**：设置为true，删除安全组默认规则

### 6. 创建随机密码

在TF文件（如main.tf）中添加以下脚本：

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
- **length**：密码长度为12
- **special**：允许使用特殊字符
- **override_special**：允许使用的特殊字符集合
- **min_lower**、**min_upper**、**min_numeric**、**min_special**：分别指定小写字母、大写字母、数字和特殊字符的最小数量

### 7. 创建GaussDB实例

在TF文件（如main.tf）中添加以下脚本：

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
  port              = "9000"
  sharding_num      = 1
  coordinator_num   = 2
  availability_zone = join(",", try(slice(data.huaweicloud_availability_zones.test.names, 0, 3), []))

  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null

  ha {
    mode             = "enterprise"
    replication_mode = "sync"
    consistency      = "eventual"
    instance_mode    = "enterprise"
  }

  volume {
    type = var.instance_volume_type
    size = var.instance_volume_size
  }
}
```

**参数说明**：
- **vpc_id**：通过引用虚拟私有云资源的ID进行赋值
- **subnet_id**：通过引用虚拟私有云子网资源的ID进行赋值
- **security_group_id**：通过引用安全组资源的ID进行赋值
- **flavor**：实例规格，固定为 `gaussdb.opengauss.ee.dn.m6.2xlarge.8.in`
- **name**：通过引用输入变量 instance_name 进行赋值
- **password**：通过引用输入变量 instance_password 进行赋值，为空时使用随机密码
- **port**：数据库端口，固定为9000
- **sharding_num**：分片数量，固定为1
- **coordinator_num**：协调节点数量，固定为2
- **availability_zone**：通过引用可用区列表数据源的前三个可用区进行赋值
- **enterprise_project_id**：通过引用输入变量 enterprise_project_id 进行赋值，为空时使用默认企业项目
- **ha.mode**：高可用模式，固定为 `enterprise`
- **ha.replication_mode**：复制模式，固定为 `sync`
- **ha.consistency**：一致性模式，固定为 `eventual`
- **ha.instance_mode**：实例模式，固定为 `enterprise`
- **volume.type**：通过引用输入变量 instance_volume_type 进行赋值
- **volume.size**：通过引用输入变量 instance_volume_size 进行赋值

### 8. 创建ASP采集任务

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ASP采集资源
variable "start_time" {
  description = "The start time for ASP collect in yyyy-mm-ddThh:mm:ssZ format"
  type        = string
}

variable "end_time" {
  description = "The end time for ASP collect in yyyy-mm-ddThh:mm:ssZ format"
  type        = string
}

resource "huaweicloud_gaussdb_asp_collect" "test" {
  instance_id = huaweicloud_gaussdb_instance.test.id
  start_time  = var.start_time
  end_time    = var.end_time
}
```

**参数说明**：
- **instance_id**：通过引用GaussDB实例资源的ID进行赋值
- **start_time**：通过引用输入变量 start_time 进行赋值，格式为 `yyyy-mm-ddThh:mm:ssZ`
- **end_time**：通过引用输入变量 end_time 进行赋值，格式为 `yyyy-mm-ddThh:mm:ssZ`

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
start_time          = "2026-08-24T09:00:00+0000"
end_time            = "2026-08-24T12:00:00+0000"
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
3. 确认资源计划无误后，运行 `terraform apply` 开始创建ASP采集任务
4. 运行 `terraform show` 查看已创建的ASP采集任务

## 参考信息

- [华为云GaussDB产品文档](https://support.huaweicloud.com/gaussdb/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [GaussDB ASP采集最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/gaussdb/gaussdb-asp-collect)
