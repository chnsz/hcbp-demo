# 部署Kafka实例配置

## 应用场景

分布式消息服务（DMS）Kafka是一款高吞吐、高可用的分布式消息中间件，广泛应用于日志采集、流式数据处理、业务解耦与削峰填谷等场景。在实际部署中，Kafka实例需要与虚拟私有云（VPC）、子网、安全组等网络资源协同配置，才能保证实例的网络可达性与访问安全。

本最佳实践将介绍如何使用Terraform自动化部署Kafka实例配置，包括创建VPC、子网、安全组，以及创建Kafka实例并配置其规格、存储、可用区与访问认证等参数。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [Kafka实例规格（data.huaweicloud_dms_kafka_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dms_kafka_flavors)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [Kafka实例（huaweicloud_dms_kafka_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafka_instance)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── data.huaweicloud_dms_kafka_flavors
        └── huaweicloud_dms_kafka_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dms_kafka_instance

huaweicloud_networking_secgroup
    └── huaweicloud_dms_kafka_instance
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用区列表

在TF文件（如main.tf）中添加以下脚本以查询当前区域下的可用区列表：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区列表
variable "availability_zones" {
  description = "The availability zones to which the Kafka instance belongs"
  type        = list(string)
  default     = []
  nullable    = false
}

data "huaweicloud_availability_zones" "test" {
  count = length(var.availability_zones) == 0 ? 1 : 0
}
```

**参数说明**：
- **count**：当输入变量 availability_zones 为空时创建该数据源，用于自动获取当前区域下的可用区列表

### 3. 创建虚拟私有云

在TF文件中添加以下脚本以创建VPC：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云
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
- **name**：VPC名称，通过引用输入变量 vpc_name 进行赋值
- **cidr**：VPC网段，通过引用输入变量 vpc_cidr 进行赋值

### 4. 创建虚拟私有云子网

在TF文件中添加以下脚本以创建子网：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云子网
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
- **vpc_id**：子网所属的VPC ID，引用前一步创建的VPC的ID
- **name**：子网名称，通过引用输入变量 subnet_name 进行赋值
- **cidr**：子网网段，当输入变量 subnet_cidr 为空时基于VPC网段自动划分
- **gateway_ip**：子网网关IP，当输入变量 subnet_gateway_ip 为空时基于子网网段自动计算

### 5. 创建安全组

在TF文件中添加以下脚本以创建安全组：

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
- **name**：安全组名称，通过引用输入变量 security_group_name 进行赋值
- **delete_default_rules**：是否删除安全组默认规则，设置为 true 表示删除默认规则

### 6. 查询Kafka实例规格

在TF文件中添加以下脚本以查询Kafka实例规格：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询Kafka实例规格
variable "instance_flavor_id" {
  description = "The flavor ID of the Kafka instance"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_flavor_type" {
  description = "The flavor type of the Kafka instance"
  type        = string
  default     = "cluster"
}

variable "instance_storage_spec_code" {
  description = "The storage specification code of the Kafka instance"
  type        = string
  default     = "dms.physical.storage.ultra.v2"
}

data "huaweicloud_dms_kafka_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  type               = var.instance_flavor_type
  availability_zones = length(var.availability_zones) == 0 ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 3)) : var.availability_zones
  storage_spec_code  = var.instance_storage_spec_code
}
```

**参数说明**：
- **count**：当输入变量 instance_flavor_id 为空时创建该数据源，用于自动查询Kafka实例规格
- **type**：规格类型，通过引用输入变量 instance_flavor_type 进行赋值
- **availability_zones**：可用区列表，当输入变量 availability_zones 为空时取查询到的可用区列表的前三个
- **storage_spec_code**：存储规格编码，通过引用输入变量 instance_storage_spec_code 进行赋值

### 7. 创建Kafka实例

在TF文件中添加以下脚本以创建Kafka实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建Kafka实例
variable "instance_name" {
  description = "The name of the Kafka instance"
  type        = string
}

variable "instance_engine_version" {
  description = "The engine version of the Kafka instance"
  type        = string
  default     = "2.7"
}

variable "instance_storage_space" {
  description = "The storage space of the Kafka instance"
  type        = number
  default     = 600
}

variable "instance_broker_num" {
  description = "The number of brokers of the Kafka instance"
  type        = number
  default     = 3
}

variable "instance_ssl_enable" {
  description = "The SSL enable of the Kafka instance"
  type        = bool
  default     = false
}

variable "instance_access_user_name" {
  description = "The access user of the Kafka instance"
  type        = string
  default     = ""
}

variable "instance_access_user_password" {
  description = "The access password of the Kafka instance"
  sensitive   = true
  type        = string
  default     = ""
}

variable "instance_description" {
  description = "The description of the Kafka instance"
  type        = string
  default     = ""
}

variable "charging_mode" {
  description = "The charging mode of the Kafka instance"
  type        = string
  default     = "postPaid"
}

variable "period_unit" {
  description = "The period unit of the Kafka instance"
  type        = string
  default     = null
}

variable "period" {
  description = "The period of the Kafka instance"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "The auto renew of the Kafka instance"
  type        = string
  default     = "false"
}

resource "huaweicloud_dms_kafka_instance" "test" {
  name               = var.instance_name
  availability_zones = length(var.availability_zones) == 0 ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 3)) : var.availability_zones
  engine_version     = var.instance_engine_version
  flavor_id          = var.instance_flavor_id == "" ? try(data.huaweicloud_dms_kafka_flavors.test[0].flavors[0].id, null) : var.instance_flavor_id
  storage_spec_code  = var.instance_storage_spec_code
  storage_space      = var.instance_storage_space
  broker_num         = var.instance_broker_num
  vpc_id             = huaweicloud_vpc.test.id
  network_id         = huaweicloud_vpc_subnet.test.id
  security_group_id  = huaweicloud_networking_secgroup.test.id
  ssl_enable         = var.instance_ssl_enable
  access_user        = var.instance_access_user_name
  password           = var.instance_access_user_password
  description        = var.instance_description
  charging_mode      = var.charging_mode
  period_unit        = var.period_unit
  period             = var.period
  auto_renew         = var.auto_renew

  # If you want to change some of the following parameters, you need to remove the corresponding fields from "lifecycle.ignore_changes".
  lifecycle {
    ignore_changes = [
      availability_zones,
      flavor_id,
    ]
  }
}
```

**参数说明**：
- **name**：Kafka实例名称，通过引用输入变量 instance_name 进行赋值
- **availability_zones**：可用区列表，当输入变量 availability_zones 为空时取查询到的可用区列表的前三个
- **engine_version**：引擎版本，通过引用输入变量 instance_engine_version 进行赋值
- **flavor_id**：规格ID，当输入变量 instance_flavor_id 为空时取查询到的规格列表中的第一个规格ID
- **storage_spec_code**：存储规格编码，通过引用输入变量 instance_storage_spec_code 进行赋值
- **storage_space**：存储空间，通过引用输入变量 instance_storage_space 进行赋值
- **broker_num**：代理数量，通过引用输入变量 instance_broker_num 进行赋值
- **vpc_id**：实例所属的VPC ID，引用前一步创建的VPC的ID
- **network_id**：实例所属的子网ID，引用前一步创建的子网的ID
- **security_group_id**：实例所属的安全组ID，引用前一步创建的安全组的ID
- **ssl_enable**：是否开启SSL，通过引用输入变量 instance_ssl_enable 进行赋值
- **access_user**：访问用户，通过引用输入变量 instance_access_user_name 进行赋值
- **password**：访问密码，通过引用输入变量 instance_access_user_password 进行赋值
- **description**：实例描述，通过引用输入变量 instance_description 进行赋值
- **charging_mode**：计费模式，通过引用输入变量 charging_mode 进行赋值
- **period_unit**：计费周期单位，通过引用输入变量 period_unit 进行赋值
- **period**：计费周期，通过引用输入变量 period 进行赋值
- **auto_renew**：是否自动续费，通过引用输入变量 auto_renew 进行赋值

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
vpc_name                      = "tf_test_instance"
subnet_name                   = "tf_test_instance"
security_group_name           = "tf_test_instance"
instance_name                 = "tf_test_instance"
instance_ssl_enable           = true
instance_access_user_name     = "admin"
instance_access_user_password = "YourKafkaInstancePassword!"
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
3. 确认资源计划无误后，运行 `terraform apply` 开始创建Kafka实例
4. 运行 `terraform show` 查看已创建的Kafka实例

## 参考信息

- [华为云分布式消息服务Kafka产品文档](https://support.huaweicloud.com/kafka/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DMS Kafka实例配置最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dms/kafka/instance-configuration)
