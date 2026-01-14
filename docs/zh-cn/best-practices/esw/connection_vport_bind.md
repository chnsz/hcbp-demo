# 部署连接与虚拟端口绑定

## 应用场景

企业交换机（Enterprise Switch，ESW）是华为云提供的高性能、高可用的企业级网络交换服务，支持大二层网络互通，实现跨可用区的二层网络连接。通过ESW连接与虚拟端口绑定，可以将ESW连接与VPC子网私有IP进行绑定，实现网络资源的灵活配置和管理。本最佳实践将介绍如何使用Terraform自动化部署ESW连接与虚拟端口绑定，包括VPC、子网、ESW实例、ESW连接、VPC子网私有IP和连接虚拟端口绑定的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [ESW规格数据源（huaweicloud_esw_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/esw_flavors)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [ESW实例资源（huaweicloud_esw_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/esw_instance)
- [ESW连接资源（huaweicloud_esw_connection）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/esw_connection)
- [VPC子网私有IP资源（huaweicloud_vpc_subnet_private_ip）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet_private_ip)
- [ESW连接虚拟端口绑定资源（huaweicloud_esw_connection_vport_bind）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/esw_connection_vport_bind)

### 资源/数据源依赖关系

```text
data.huaweicloud_esw_flavors
    └── huaweicloud_esw_instance

huaweicloud_vpc
    ├── huaweicloud_vpc_subnet
    │   ├── huaweicloud_esw_instance
    │   ├── huaweicloud_esw_connection
    │   └── huaweicloud_vpc_subnet_private_ip
    │       └── huaweicloud_esw_connection_vport_bind
    └── huaweicloud_esw_instance

huaweicloud_esw_instance
    └── huaweicloud_esw_connection
        └── huaweicloud_esw_connection_vport_bind
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询ESW规格数据源

在TF文件（如main.tf）中添加以下脚本以查询ESW规格信息：

```hcl
# 查询ESW规格信息
data "huaweicloud_esw_flavors" "test" {}
```

**参数说明**：
- 该数据源无需参数，会自动查询当前区域可用的ESW规格信息

### 3. 创建VPC资源

在TF文件（如main.tf）中添加以下脚本以创建VPC：

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

# 创建VPC
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量vpc_name进行赋值
- **cidr**：VPC的CIDR地址块，通过引用输入变量vpc_cidr进行赋值，默认值为"192.168.0.0/16"

### 4. 创建VPC子网资源

在TF文件（如main.tf）中添加以下脚本以创建多个VPC子网（本实践需要2个子网）：

```hcl
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

# 创建VPC子网（创建2个子网）
resource "huaweicloud_vpc_subnet" "test" {
  count = 2

  vpc_id     = huaweicloud_vpc.test.id
  name       = format("%s-%d", var.subnet_name, count.index)
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**参数说明**：
- **count**：创建数量，设置为2，创建2个子网
- **vpc_id**：VPC ID，通过引用VPC资源进行赋值
- **name**：子网名称，通过引用输入变量subnet_name和count索引进行赋值，格式为"subnet_name-0"和"subnet_name-1"
- **cidr**：子网的CIDR地址块，通过引用输入变量或自动计算进行赋值
- **gateway_ip**：子网网关IP，通过引用输入变量或自动计算进行赋值

### 5. 创建ESW实例资源

在TF文件（如main.tf）中添加以下脚本以创建ESW实例：

```hcl
variable "esw_instance_name" {
  description = "The name of the instance"
  type        = string
  default     = ""
}

variable "esw_instance_ha_mode" {
  description = "The HA mode of the instance"
  type        = string
  default     = ""
}

variable "esw_instance_description" {
  description = "The description of the instance"
  type        = string
  default     = ""
}

variable "esw_instance_tunnel_ip" {
  description = "The tunnel IP of the instance"
  type        = string
  default     = "192.168.0.192"
}

# 创建ESW实例
resource "huaweicloud_esw_instance" "test" {
  name        = var.esw_instance_name
  flavor_ref  = try(data.huaweicloud_esw_flavors.test.flavors[0].name, "")
  ha_mode     = var.esw_instance_ha_mode
  description = var.esw_instance_description

  availability_zones {
    primary = try(data.huaweicloud_esw_flavors.test.flavors.0.available_zones[0], "")
    standby = try(data.huaweicloud_esw_flavors.test.flavors.0.available_zones[1], "")
  }

  tunnel_info {
    vpc_id       = huaweicloud_vpc.test.id
    virsubnet_id = huaweicloud_vpc_subnet.test[0].id
    tunnel_ip    = var.esw_instance_tunnel_ip
  }

  charge_infos {
    charge_mode = "postPaid"
  }
}
```

**参数说明**：
- **name**：ESW实例名称，通过引用输入变量esw_instance_name进行赋值
- **flavor_ref**：规格引用，通过引用ESW规格数据源进行赋值
- **ha_mode**：高可用模式，通过引用输入变量esw_instance_ha_mode进行赋值，可选参数
- **description**：实例描述，通过引用输入变量esw_instance_description进行赋值，可选参数
- **availability_zones.primary**：主可用区，通过引用ESW规格数据源进行赋值
- **availability_zones.standby**：备可用区，通过引用ESW规格数据源进行赋值
- **tunnel_info.vpc_id**：隧道VPC ID，通过引用VPC资源进行赋值
- **tunnel_info.virsubnet_id**：隧道虚拟子网ID，通过引用第一个子网资源进行赋值
- **tunnel_info.tunnel_ip**：隧道IP，通过引用输入变量esw_instance_tunnel_ip进行赋值，默认值为"192.168.0.192"
- **charge_infos.charge_mode**：计费模式，设置为"postPaid"（按需计费）

### 6. 创建ESW连接资源

在TF文件（如main.tf）中添加以下脚本以创建ESW连接：

```hcl
variable "esw_connection_name" {
  description = "The name of the connection"
  type        = string
  default     = ""
}

variable "esw_connection_fixed_ips" {
  description = "The fixed IP IP of the connection"
  type        = list(string)
  default     = []
}

variable "esw_connection_segmentation_id" {
  description = "The segmentation ID of the connection"
  type        = number
  default     = 10000
}

# 创建ESW连接
resource "huaweicloud_esw_connection" "test" {
  instance_id  = huaweicloud_esw_instance.test.id
  name         = var.esw_connection_name
  vpc_id       = huaweicloud_vpc.test.id
  virsubnet_id = huaweicloud_vpc_subnet.test[1].id
  fixed_ips    = var.esw_connection_fixed_ips

  remote_infos {
    segmentation_id = var.esw_connection_segmentation_id
    tunnel_ip       = huaweicloud_esw_instance.test.tunnel_info[0].tunnel_ip
    tunnel_port     = huaweicloud_esw_instance.test.tunnel_info[0].tunnel_port
  }
}
```

**参数说明**：
- **instance_id**：ESW实例ID，通过引用ESW实例资源进行赋值
- **name**：连接名称，通过引用输入变量esw_connection_name进行赋值，可选参数
- **vpc_id**：VPC ID，通过引用VPC资源进行赋值
- **virsubnet_id**：虚拟子网ID，通过引用第二个子网资源进行赋值
- **fixed_ips**：固定IP列表，通过引用输入变量esw_connection_fixed_ips进行赋值，可选参数
- **remote_infos.segmentation_id**：分段ID，通过引用输入变量esw_connection_segmentation_id进行赋值，默认值为10000
- **remote_infos.tunnel_ip**：隧道IP，通过引用ESW实例的隧道IP进行赋值
- **remote_infos.tunnel_port**：隧道端口，通过引用ESW实例的隧道端口进行赋值

### 7. 创建VPC子网私有IP资源

在TF文件（如main.tf）中添加以下脚本以创建VPC子网私有IP：

```hcl
# 创建VPC子网私有IP
resource "huaweicloud_vpc_subnet_private_ip" "test" {
  subnet_id    = huaweicloud_vpc_subnet.test[1].id
  device_owner = "neutron:VIP_PORT"
}
```

**参数说明**：
- **subnet_id**：子网ID，通过引用第二个子网资源进行赋值
- **device_owner**：设备所有者，设置为"neutron:VIP_PORT"（虚拟IP端口）

### 8. 创建ESW连接虚拟端口绑定资源

在TF文件（如main.tf）中添加以下脚本以创建ESW连接与虚拟端口绑定：

```hcl
# 创建ESW连接虚拟端口绑定
resource "huaweicloud_esw_connection_vport_bind" "test" {
  connection_id = huaweicloud_esw_connection.test.id
  port_id       = huaweicloud_vpc_subnet_private_ip.test.id
}
```

**参数说明**：
- **connection_id**：连接ID，通过引用ESW连接资源进行赋值
- **port_id**：端口ID，通过引用VPC子网私有IP资源进行赋值

### 9. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# VPC和子网配置
vpc_name    = "tf_test_esw_connection_cport_bind"
vpc_cidr    = "192.168.0.0/16"
subnet_name = "tf_test_esw_connection_cport_bind"

# ESW实例配置
esw_instance_name    = "tf_test_esw_connection_cport_bind"
esw_instance_ha_mode = "ha"

# ESW连接配置
esw_connection_name            = "tf_test_esw_connection_cport_bind"
esw_connection_segmentation_id = 9999
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值，特别是：
   - `esw_instance_ha_mode`可以设置为"ha"（高可用模式）或其他模式
   - `esw_connection_segmentation_id`需要设置分段ID，用于网络分段标识
   - `esw_connection_fixed_ips`可以设置连接的固定IP列表，可选参数
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="esw_instance_name=my_instance" -var="vpc_name=my_vpc"`
2. 环境变量：`export TF_VAR_esw_instance_name=my_instance` 和 `export TF_VAR_vpc_name=my_vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。ESW实例需要配置主备可用区，确保高可用性。ESW连接需要配置远程信息，包括分段ID、隧道IP和隧道端口。

### 10. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建ESW连接与虚拟端口绑定：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建VPC、子网、ESW实例、ESW连接、VPC子网私有IP和连接虚拟端口绑定
4. 运行 `terraform show` 查看已创建的连接虚拟端口绑定详情

> 注意：ESW实例创建后需要等待实例状态变为可用，才能继续创建ESW连接。ESW连接创建后，才能创建连接虚拟端口绑定。确保VPC和子网已正确创建，并且子网CIDR配置正确。

## 参考信息

- [华为云ESW产品文档](https://support.huaweicloud.com/esw/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [连接与虚拟端口绑定最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/esw/connection-vport-bind)
