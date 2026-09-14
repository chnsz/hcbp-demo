# 部署Kafka公网访问实例网络

## 应用场景

分布式消息服务（DMS）Kafka 是华为云提供的高吞吐、高可靠的消息中间件服务，广泛应用于日志采集、流式数据处理和业务解耦等场景。默认情况下，Kafka 实例仅支持 VPC 内网访问，当业务客户端位于本地数据中心、其他 VPC 或公网环境时，需要为实例配置公网访问能力。

本最佳实践将介绍如何使用Terraform自动化部署支持公网访问的Kafka实例网络配置，包括VPC、子网、安全组、弹性公网IP（EIP）以及Kafka实例的公网访问协议配置。通过为每个Broker绑定EIP并开启相应的公网访问协议，外部客户端即可通过公网安全地连接Kafka实例。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [Kafka实例规格（data.huaweicloud_dms_kafka_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dms_kafka_flavors)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [弹性公网IP（huaweicloud_vpc_eip）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [Kafka实例（huaweicloud_dms_kafka_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafka_instance)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_dms_kafka_instance

data.huaweicloud_dms_kafka_flavors
    └── huaweicloud_dms_kafka_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
            └── huaweicloud_dms_kafka_instance

huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule
    └── huaweicloud_dms_kafka_instance

huaweicloud_vpc_eip
    └── huaweicloud_dms_kafka_instance
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用区列表

在TF文件（如main.tf）中添加以下脚本以查询Kafka实例可用的可用区：

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
- **count**：当输入变量 availability_zones 为空时执行查询，否则使用用户指定的可用区列表

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
- **name**：通过引用输入变量 vpc_name 进行赋值
- **cidr**：通过引用输入变量 vpc_cidr 进行赋值，默认为 192.168.0.0/16

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
- **vpc_id**：通过引用 huaweicloud_vpc.test.id 进行赋值，关联上一步创建的VPC
- **name**：通过引用输入变量 subnet_name 进行赋值
- **cidr**：通过引用输入变量 subnet_cidr 进行赋值，为空时基于VPC网段自动划分子网
- **gateway_ip**：通过引用输入变量 subnet_gateway_ip 进行赋值，为空时自动计算网关IP

### 5. 创建安全组及安全组规则

在TF文件中添加以下脚本以创建安全组并放通Kafka公网访问端口：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组及安全组规则
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

variable "security_group_rule_ports" {
  description = "The ports of the security group rule"
  type        = string
  default     = "9094,9095"
}

variable "security_group_rule_remote_ip_prefix" {
  description = "The remote IP prefix of the security group rule"
  type        = string
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
  remote_ip_prefix  = var.security_group_rule_remote_ip_prefix
}
```

**参数说明**：
- **name**：通过引用输入变量 security_group_name 进行赋值
- **delete_default_rules**：设置为 true，删除安全组默认规则
- **security_group_id**：通过引用 huaweicloud_networking_secgroup.test.id 进行赋值
- **direction**：设置为 ingress，表示入方向规则
- **protocol**：设置为 tcp，表示TCP协议
- **ports**：通过引用输入变量 security_group_rule_ports 进行赋值，默认为 9094,9095，分别对应公网明文访问端口和公网加密访问端口
- **remote_ip_prefix**：通过引用输入变量 security_group_rule_remote_ip_prefix 进行赋值，指定允许访问Kafka实例的客户端IP地址或地址段

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
  availability_zones = length(var.availability_zones) == 0 ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1)) : var.availability_zones
  storage_spec_code  = var.instance_storage_spec_code
}
```

**参数说明**：
- **count**：当输入变量 instance_flavor_id 为空时执行查询，否则使用用户指定的规格ID
- **type**：通过引用输入变量 instance_flavor_type 进行赋值，默认为 cluster
- **availability_zones**：当输入变量 availability_zones 为空时取可用区列表的第一个可用区，否则使用用户指定的可用区列表
- **storage_spec_code**：通过引用输入变量 instance_storage_spec_code 进行赋值

### 7. 创建弹性公网IP

在TF文件中添加以下脚本以为每个Broker创建弹性公网IP：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建弹性公网IP
variable "instance_broker_num" {
  description = "The number of brokers of the Kafka instance"
  type        = number
  default     = 3
}

variable "eip_type" {
  description = "The type of the EIP"
  type        = string
  default     = "5_bgp"
}

variable "bandwidth_name" {
  description = "The name of the bandwidth"
  type        = string
}

variable "bandwidth_size" {
  description = "The size of the bandwidth"
  type        = number
  default     = 5
}

variable "bandwidth_share_type" {
  description = "The share type of the bandwidth"
  type        = string
  default     = "PER"
}

variable "bandwidth_charge_mode" {
  description = "The charge mode of the bandwidth"
  type        = string
  default     = "traffic"
}

resource "huaweicloud_vpc_eip" "test" {
  count = var.instance_broker_num

  publicip {
    type = var.eip_type
  }

  bandwidth {
    name        = var.bandwidth_name
    size        = var.bandwidth_size
    share_type  = var.bandwidth_share_type
    charge_mode = var.bandwidth_charge_mode
  }
}
```

**参数说明**：
- **count**：通过引用输入变量 instance_broker_num 进行赋值，为每个Broker创建一个EIP，数量必须与Broker数量一致
- **publicip.type**：通过引用输入变量 eip_type 进行赋值，默认为 5_bgp
- **bandwidth.name**：通过引用输入变量 bandwidth_name 进行赋值
- **bandwidth.size**：通过引用输入变量 bandwidth_size 进行赋值，默认为 5
- **bandwidth.share_type**：通过引用输入变量 bandwidth_share_type 进行赋值，默认为 PER
- **bandwidth.charge_mode**：通过引用输入变量 bandwidth_charge_mode 进行赋值，默认为 traffic

### 8. 创建Kafka实例并配置公网访问

在TF文件中添加以下脚本以创建Kafka实例并开启公网访问协议：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建Kafka实例并配置公网访问
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

variable "instance_description" {
  description = "The description of the Kafka instance"
  type        = string
  default     = ""
}

variable "instance_access_user_name" {
  description = "The access user of the Kafka instance"
  type        = string
  default     = null
}

variable "instance_access_user_password" {
  description = "The access password of the Kafka instance"
  type        = string
  sensitive   = true
  default     = null
}

variable "instance_enabled_mechanisms" {
  description = "The enabled mechanisms of the Kafka instance"
  type        = list(string)
  default     = null
}

variable "instance_public_plain_enable" {
  description = "Whether to enable public plaintext access"
  type        = bool
  default     = true
}

variable "instance_public_sasl_ssl_enable" {
  description = "Whether to enable public SASL SSL access"
  type        = bool
  default     = false
}

variable "instance_public_sasl_plaintext_enable" {
  description = "Whether to enable public SASL plaintext access"
  type        = bool
  default     = false
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
  description        = var.instance_description
  public_ip_ids      = huaweicloud_vpc_eip.test[*].id
  access_user        = var.instance_access_user_name
  password           = var.instance_access_user_password
  enabled_mechanisms = var.instance_enabled_mechanisms

  port_protocol {
    private_plain_enable         = true
    public_plain_enable          = var.instance_public_plain_enable
    public_sasl_ssl_enable       = var.instance_public_sasl_ssl_enable
    public_sasl_plaintext_enable = var.instance_public_sasl_plaintext_enable
  }

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
- **name**：通过引用输入变量 instance_name 进行赋值
- **availability_zones**：当输入变量 availability_zones 为空时取可用区列表的前三个可用区，否则使用用户指定的可用区列表
- **engine_version**：通过引用输入变量 instance_engine_version 进行赋值，默认为 2.7
- **flavor_id**：当输入变量 instance_flavor_id 为空时取查询到的第一个规格ID，否则使用用户指定的规格ID
- **storage_spec_code**：通过引用输入变量 instance_storage_spec_code 进行赋值
- **storage_space**：通过引用输入变量 instance_storage_space 进行赋值，默认为 600
- **broker_num**：通过引用输入变量 instance_broker_num 进行赋值，默认为 3
- **vpc_id**：通过引用 huaweicloud_vpc.test.id 进行赋值
- **network_id**：通过引用 huaweicloud_vpc_subnet.test.id 进行赋值
- **security_group_id**：通过引用 huaweicloud_networking_secgroup.test.id 进行赋值
- **description**：通过引用输入变量 instance_description 进行赋值
- **public_ip_ids**：通过引用 huaweicloud_vpc_eip.test[*].id 进行赋值，为每个Broker绑定一个EIP
- **access_user**：通过引用输入变量 instance_access_user_name 进行赋值，用于SASL认证
- **password**：通过引用输入变量 instance_access_user_password 进行赋值，用于SASL认证
- **enabled_mechanisms**：通过引用输入变量 instance_enabled_mechanisms 进行赋值，可选值为 PLAIN 和 SCRAM-SHA-512
- **port_protocol.private_plain_enable**：设置为 true，开启内网明文访问
- **port_protocol.public_plain_enable**：通过引用输入变量 instance_public_plain_enable 进行赋值，控制是否开启公网明文访问（端口9094）
- **port_protocol.public_sasl_ssl_enable**：通过引用输入变量 instance_public_sasl_ssl_enable 进行赋值，控制是否开启公网SASL SSL访问（端口9095）
- **port_protocol.public_sasl_plaintext_enable**：通过引用输入变量 instance_public_sasl_plaintext_enable 进行赋值，控制是否开启公网SASL明文访问（端口9094）

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
vpc_name                             = "tf_test_kafka_instance"
subnet_name                          = "tf_test_kafka_instance"
security_group_name                  = "tf_test_kafka_instance"
security_group_rule_remote_ip_prefix = "your_client_ip_address"
instance_name                        = "tf_test_kafka_instance"
bandwidth_name                       = "tf_test_kafka_instance_bandwidth"
instance_access_user_name            = "admin"
instance_access_user_password        = "yourInstanceAccessPassword!"
instance_enabled_mechanisms          = ["SCRAM-SHA-512"]
instance_public_sasl_ssl_enable      = true
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
3. 确认资源计划无误后，运行 `terraform apply` 开始创建支持公网访问的Kafka实例
4. 运行 `terraform show` 查看已创建的支持公网访问的Kafka实例

## 参考信息

- [华为云分布式消息服务Kafka产品文档](https://support.huaweicloud.com/kafka/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DMS Kafka公网访问实例网络最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dms/kafka/public-access-instance-network)
