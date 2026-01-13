# 部署Kafka公网访问实例网络

## 应用场景

华为云分布式消息服务Kafka版支持通过公网访问实例，适用于需要在公网环境下访问Kafka服务的场景，如跨地域访问、开发测试环境等。通过配置公网访问，可以为Kafka实例绑定弹性公网IP（EIP），并配置相应的安全组规则和端口协议，实现安全的公网访问。本最佳实践将介绍如何使用Terraform自动化部署支持公网访问的Kafka实例网络配置，包括VPC、子网、安全组、EIP和Kafka实例的公网访问配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区数据源（huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [Kafka规格数据源（huaweicloud_dms_kafka_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dms_kafka_flavors)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则资源（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [弹性公网IP资源（huaweicloud_vpc_eip）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [Kafka实例资源（huaweicloud_dms_kafka_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dms_kafka_instance)

### 资源/数据源依赖关系

```text
data.huaweicloud_availability_zones
    └── data.huaweicloud_dms_kafka_flavors
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
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询数据源

在TF文件（如main.tf）中添加以下脚本以查询可用区和Kafka规格信息：

```hcl
# 查询可用区信息
data "huaweicloud_availability_zones" "test" {
  count = length(var.availability_zones) == 0 ? 1 : 0
}

# 查询Kafka规格信息
data "huaweicloud_dms_kafka_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  type               = var.instance_flavor_type
  availability_zones = length(var.availability_zones) == 0 ? try(slice(data.huaweicloud_availability_zones.test[0].names, 0, 1)) : var.availability_zones
  storage_spec_code  = var.instance_storage_spec_code
}
```

**参数说明**：
- **type**：规格类型，通过引用输入变量instance_flavor_type进行赋值，默认值为"cluster"（集群模式）
- **availability_zones**：可用区列表，通过引用输入变量availability_zones或可用区数据源进行赋值
- **storage_spec_code**：存储规格代码，通过引用输入变量instance_storage_spec_code进行赋值，默认值为"dms.physical.storage.ultra.v2"

### 3. 创建基础网络资源

在TF文件（如main.tf）中添加以下脚本以创建VPC、子网和安全组：

```hcl
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  default     = ""
}

variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

# 创建VPC
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}

# 创建子网
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}

# 创建安全组
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

### 4. 创建安全组规则

在TF文件（如main.tf）中添加以下脚本以创建安全组规则，允许公网访问Kafka实例的端口：

```hcl
variable "security_group_rule_ports" {
  description = "The ports of the security group rule"
  type        = string
  default     = "9094,9095"
}

variable "security_group_rule_remote_ip_prefix" {
  description = "The remote IP prefix of the security group rule"
  type        = string
}

# 创建安全组规则
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
- **security_group_id**：安全组ID，通过引用安全组资源进行赋值
- **direction**：规则方向，设置为"ingress"（入方向）
- **ethertype**：IP协议类型，设置为"IPv4"
- **protocol**：协议类型，设置为"tcp"
- **ports**：端口范围，通过引用输入变量security_group_rule_ports进行赋值，默认值为"9094,9095"（Kafka公网访问端口）
- **remote_ip_prefix**：远端IP地址段，通过引用输入变量security_group_rule_remote_ip_prefix进行赋值，用于限制允许访问的IP范围

### 5. 创建弹性公网IP

在TF文件（如main.tf）中添加以下脚本以创建弹性公网IP（EIP），用于绑定到Kafka实例：

```hcl
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

# 创建弹性公网IP（根据broker数量创建对应数量的EIP）
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
- **count**：创建数量，通过引用输入变量instance_broker_num进行赋值，确保为每个broker创建一个EIP
- **publicip.type**：公网IP类型，通过引用输入变量eip_type进行赋值，默认值为"5_bgp"（全动态BGP）
- **bandwidth.name**：带宽名称，通过引用输入变量bandwidth_name进行赋值
- **bandwidth.size**：带宽大小，通过引用输入变量bandwidth_size进行赋值，默认值为5（Mbit/s）
- **bandwidth.share_type**：带宽共享类型，通过引用输入变量bandwidth_share_type进行赋值，默认值为"PER"（独享）
- **bandwidth.charge_mode**：带宽计费模式，通过引用输入变量bandwidth_charge_mode进行赋值，默认值为"traffic"（按流量计费）

### 6. 创建Kafka实例资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建支持公网访问的Kafka实例资源：

```hcl
variable "availability_zones" {
  description = "The availability zones to which the Kafka instance belongs"
  type        = list(string)
  default     = []
}

variable "instance_name" {
  description = "The name of the Kafka instance"
  type        = string
}

variable "instance_engine_version" {
  description = "The engine version of the Kafka instance"
  type        = string
  default     = "2.7"
}

variable "instance_flavor_id" {
  description = "The flavor ID of the Kafka instance"
  type        = string
  default     = ""
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
  description = "The access user name of the Kafka instance"
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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建支持公网访问的Kafka实例资源
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

  lifecycle {
    ignore_changes = [
      availability_zones,
      flavor_id,
    ]
  }
}
```

**参数说明**：
- **name**：Kafka实例名称，通过引用输入变量instance_name进行赋值
- **availability_zones**：可用区列表，通过引用输入变量availability_zones或可用区数据源进行赋值
- **engine_version**：引擎版本，通过引用输入变量instance_engine_version进行赋值，默认值为"2.7"
- **flavor_id**：规格ID，通过引用输入变量instance_flavor_id或Kafka规格数据源进行赋值
- **storage_spec_code**：存储规格代码，通过引用输入变量instance_storage_spec_code进行赋值，默认值为"dms.physical.storage.ultra.v2"
- **storage_space**：存储空间，通过引用输入变量instance_storage_space进行赋值，默认值为600（GB）
- **broker_num**：Broker数量，通过引用输入变量instance_broker_num进行赋值，默认值为3
- **vpc_id**：VPC ID，通过引用VPC资源进行赋值
- **network_id**：网络子网ID，通过引用子网资源进行赋值
- **security_group_id**：安全组ID，通过引用安全组资源进行赋值
- **description**：实例描述，通过引用输入变量instance_description进行赋值，可选参数，默认值为空字符串
- **public_ip_ids**：公网IP ID列表，通过引用EIP资源列表进行赋值，用于为Kafka实例绑定公网IP
- **access_user**：访问用户名，通过引用输入变量instance_access_user_name进行赋值，可选参数，默认值为null
- **password**：访问密码，通过引用输入变量instance_access_user_password进行赋值，可选参数，默认值为null
- **enabled_mechanisms**：启用的认证机制，通过引用输入变量instance_enabled_mechanisms进行赋值，可选参数，默认值为null，支持"SCRAM-SHA-512"等
- **port_protocol.private_plain_enable**：是否启用私网明文访问，设置为true
- **port_protocol.public_plain_enable**：是否启用公网明文访问，通过引用输入变量instance_public_plain_enable进行赋值，默认值为true
- **port_protocol.public_sasl_ssl_enable**：是否启用公网SASL SSL访问，通过引用输入变量instance_public_sasl_ssl_enable进行赋值，默认值为false
- **port_protocol.public_sasl_plaintext_enable**：是否启用公网SASL明文访问，通过引用输入变量instance_public_sasl_plaintext_enable进行赋值，默认值为false

### 7. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# VPC和子网配置
vpc_name    = "tf_test_kafka_instance"
vpc_cidr    = "192.168.0.0/16"
subnet_name = "tf_test_kafka_instance"

# 安全组配置
security_group_name                  = "tf_test_kafka_instance"
security_group_rule_remote_ip_prefix = "your_client_ip_address"

# Kafka实例配置
instance_name                 = "tf_test_kafka_instance"
bandwidth_name                = "tf_test_kafka_instance_bandwidth"
instance_access_user_name     = "admin"
instance_access_user_password = "yourInstanceAccessPassword!"
instance_enabled_mechanisms   = ["SCRAM-SHA-512"]
instance_public_sasl_ssl_enable = true
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值，特别是：
   - `security_group_rule_remote_ip_prefix`需要设置为允许访问的客户端IP地址段（如"0.0.0.0/0"表示允许所有IP访问，但建议设置为具体的IP段以提高安全性）
   - `instance_access_user_password`需要设置符合密码复杂度要求的密码
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="instance_name=my_kafka" -var="vpc_name=my_vpc"`
2. 环境变量：`export TF_VAR_instance_name=my_kafka` 和 `export TF_VAR_vpc_name=my_vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。由于instance_access_user_password包含敏感信息，建议使用环境变量或加密的变量文件进行设置。另外，为了安全起见，建议将security_group_rule_remote_ip_prefix设置为具体的客户端IP地址段，而不是"0.0.0.0/0"。

### 8. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建支持公网访问的Kafka实例：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建Kafka实例及相关资源
4. 运行 `terraform show` 查看已创建的Kafka实例详情

> 注意：Kafka实例创建后，会为每个broker绑定一个EIP，可以通过公网IP访问Kafka服务。建议启用SASL SSL访问方式以提高安全性。实例的可用区和规格ID在创建后不能修改，因此需要在创建时正确配置。通过lifecycle.ignore_changes可以避免Terraform在后续更新时修改这些不可变参数。

## 参考信息

- [华为云分布式消息服务Kafka产品文档](https://support.huaweicloud.com/kafka/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [公网访问实例网络最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dms/kafka/public-access-instance-network)
