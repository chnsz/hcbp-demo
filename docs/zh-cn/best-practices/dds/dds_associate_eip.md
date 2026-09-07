# 部署DDS实例绑定弹性公网IP

## 应用场景

文档数据库服务（Document Database Service，DDS）是华为云提供的高性能、高可靠、高安全性的分布式文档数据库服务，完全兼容MongoDB协议，适用于多种业务场景。在实际业务中，用户可能需要从公网访问DDS实例，例如进行数据迁移、远程运维或业务系统对接等场景。

本最佳实践将介绍如何使用Terraform创建DDS实例并为其绑定弹性公网IP（EIP），实现从公网安全访问DDS实例的能力。通过本实践，您可以了解如何利用Terraform自动化部署VPC、子网、安全组、EIP和DDS实例等资源，并完成DDS实例与EIP的关联配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用分区（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DDS实例信息（data.huaweicloud_dds_instances）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dds_instances)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [弹性公网IP（huaweicloud_vpc_eip）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [DDS实例（huaweicloud_dds_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dds_instance)
- [DDS实例EIP绑定（huaweicloud_dds_instance_eip_associate）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dds_instance_eip_associate)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_dds_instance
huaweicloud_vpc
    └── huaweicloud_vpc_subnet
huaweicloud_vpc_subnet
    └── huaweicloud_dds_instance
huaweicloud_networking_secgroup
    └── huaweicloud_dds_instance
huaweicloud_vpc_eip
    └── huaweicloud_dds_instance_eip_associate
huaweicloud_dds_instance
    ├── data.huaweicloud_dds_instances
    └── huaweicloud_dds_instance_eip_associate
data.huaweicloud_dds_instances
    └── local.nodeId
local.nodeId
    └── huaweicloud_dds_instance_eip_associate
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用分区

在TF文件（如main.tf）中添加以下脚本，用于在未指定可用分区时查询可用的可用分区列表：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用分区
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**参数说明**：
- **count**：当输入变量availability_zone为空字符串时，创建该数据源查询可用分区，否则不创建。

### 3. 创建虚拟私有云VPC

在TF文件（如main.tf）中添加以下脚本，创建VPC：

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
- **name**：通过引用输入变量vpc_name进行赋值，指定VPC名称。
- **cidr**：通过引用输入变量vpc_cidr进行赋值，指定VPC的CIDR网段。

### 4. 创建子网

在TF文件（如main.tf）中添加以下脚本，创建子网：

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
- **vpc_id**：通过引用已创建的VPC资源huaweicloud_vpc.test的id进行赋值，指定子网所属的VPC。
- **name**：通过引用输入变量subnet_name进行赋值，指定子网名称。
- **cidr**：通过引用输入变量subnet_cidr进行赋值，当未指定时自动从VPC的CIDR中划分一个子网段。
- **gateway_ip**：通过引用输入变量subnet_gateway_ip进行赋值，当未指定时自动计算子网的网关地址。

### 5. 创建安全组

在TF文件（如main.tf）中添加以下脚本，创建安全组：

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
- **name**：通过引用输入变量security_group_name进行赋值，指定安全组名称。
- **delete_default_rules**：设置为true，删除安全组默认规则，便于后续自定义规则。

### 6. 创建弹性公网IP

在TF文件（如main.tf）中添加以下脚本，创建弹性公网IP：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建弹性公网IP
variable "eip_type" {
  description = "The type of the EIP"
  type        = string
  default     = "5_bgp"
}

variable "eip_bandwidth_name" {
  description = "The name of the EIP bandwidth"
  type        = string
}

variable "eip_bandwidth_size" {
  description = "The size of the EIP bandwidth in Mbit/s"
  type        = number
  default     = 5
}

variable "eip_bandwidth_charge_mode" {
  description = "The charge mode of the EIP bandwidth"
  type        = string
  default     = "traffic"
}

resource "huaweicloud_vpc_eip" "test" {
  publicip {
    type = var.eip_type
  }

  bandwidth {
    name        = var.eip_bandwidth_name
    share_type  = "PER"
    size        = var.eip_bandwidth_size
    charge_mode = var.eip_bandwidth_charge_mode
  }
}
```

**参数说明**：
- **publicip.type**：通过引用输入变量eip_type进行赋值，指定EIP的类型。
- **bandwidth.name**：通过引用输入变量eip_bandwidth_name进行赋值，指定带宽名称。
- **bandwidth.share_type**：固定为"PER"，表示独享带宽。
- **bandwidth.size**：通过引用输入变量eip_bandwidth_size进行赋值，指定带宽大小。
- **bandwidth.charge_mode**：通过引用输入变量eip_bandwidth_charge_mode进行赋值，指定带宽计费模式。

### 7. 创建DDS实例

在TF文件（如main.tf）中添加以下脚本，创建DDS实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DDS实例
variable "availability_zone" {
  description = "The availability zone to which the DDS instance belongs"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_name" {
  description = "The name of the DDS instance"
  type        = string
}

variable "instance_mode" {
  description = "The type of the DDS instance"
  type        = string
  default     = "ReplicaSet"
}

variable "database_type" {
  description = "The database version type of the DDS instance"
  type        = string
  default     = "DDS-Community"
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

variable "node_type" {
  description = "The type of the DDS instance node"
  type        = string
  default     = "replica"
}

variable "node_number" {
  description = "The number of nodes of the DDS instance"
  type        = number
  default     = 3
}

variable "node_spec_code" {
  description = "The spec code of the DDS instance node"
  type        = string
  default     = "dds.mongodb.s6.large.2.repset"
  nullable    = false
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

resource "huaweicloud_dds_instance" "test" {
  name              = var.instance_name
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id
  mode              = var.instance_mode

  datastore {
    type           = var.database_type
    version        = var.database_version
    storage_engine = var.storage_engine
  }

  flavor {
    type      = var.node_type
    num       = var.node_number
    spec_code = var.node_spec_code
    storage   = var.node_storage_type
    size      = var.node_size
    node_list = var.node_list
  }
}
```

**参数说明**：
- **name**：通过引用输入变量instance_name进行赋值，指定DDS实例名称。
- **availability_zone**：通过引用输入变量availability_zone进行赋值，当未指定时使用查询到的第一个可用分区。
- **vpc_id**：通过引用已创建的VPC资源huaweicloud_vpc.test的id进行赋值。
- **subnet_id**：通过引用已创建的子网资源huaweicloud_vpc_subnet.test的id进行赋值。
- **security_group_id**：通过引用已创建的安全组资源huaweicloud_networking_secgroup.test的id进行赋值。
- **mode**：通过引用输入变量instance_mode进行赋值，指定实例类型。
- **datastore**：配置数据库类型、版本和存储引擎，分别通过引用输入变量database_type、database_version和storage_engine进行赋值。
- **flavor**：配置节点类型、数量、规格、存储类型、磁盘大小和待删除节点列表，分别通过引用输入变量node_type、node_number、node_spec_code、node_storage_type、node_size和node_list进行赋值。

### 8. 查询DDS实例信息

在TF文件（如main.tf）中添加以下脚本，查询已创建的DDS实例信息，用于获取主节点ID：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询DDS实例信息
data "huaweicloud_dds_instances" "test" {
  name = huaweicloud_dds_instance.test.name

  depends_on = [huaweicloud_dds_instance.test]
}

locals {
  nodeId = try([for v in flatten(data.huaweicloud_dds_instances.test.instances[*].groups[*].nodes) : v if v.role == "Primary"][0].id, "")
}
```

**参数说明**：
- **name**：通过引用已创建的DDS实例huaweicloud_dds_instance.test的name进行赋值，指定查询的实例名称。
- **depends_on**：显式依赖DDS实例资源，确保实例创建完成后再查询。
- **locals.nodeId**：从查询结果中提取角色为Primary的节点ID，用于后续绑定EIP。

### 9. 绑定弹性公网IP到DDS实例

在TF文件（如main.tf）中添加以下脚本，将弹性公网IP绑定到DDS实例的主节点：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下将弹性公网IP绑定到DDS实例
resource "huaweicloud_dds_instance_eip_associate" "test" {
  instance_id = huaweicloud_dds_instance.test.id
  node_id     = local.nodeId
  public_ip   = huaweicloud_vpc_eip.test.address
}
```

**参数说明**：
- **instance_id**：通过引用已创建的DDS实例huaweicloud_dds_instance.test的id进行赋值，指定绑定的实例。
- **node_id**：通过引用本地变量local.nodeId进行赋值，指定绑定的节点ID。
- **public_ip**：通过引用已创建的弹性公网IP资源huaweicloud_vpc_eip.test的address进行赋值，指定绑定的公网IP地址。

### 10. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 根据脚本变量填写；敏感信息使用占位符
vpc_name            = "example-vpc"
subnet_name         = "example-subnet"
security_group_name = "example-security-group"
eip_bandwidth_name  = "example-bandwidth"
instance_name       = "example-dds-instance"
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
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DDS实例并绑定弹性公网IP
4. 运行 `terraform show` 查看已创建的DDS实例和弹性公网IP绑定关系

## 参考信息

- [华为云文档数据库服务产品文档](https://support.huaweicloud.com/dds/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DDS实例绑定弹性公网IP最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dds/dds-associate-eip)
