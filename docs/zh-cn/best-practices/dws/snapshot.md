# 部署快照

## 应用场景

数据仓库服务（DWS）是华为云提供的在线联机分析处理（OLAP）企业级数据仓库服务，面向海量数据分析场景提供高性能、高可靠、易扩展的数据仓库能力。快照是DWS集群数据保护的重要手段，可在集群发生故障或数据误操作时，将集群数据恢复到指定快照点，保障数据仓库业务的连续性与数据安全。

本最佳实践将介绍如何使用Terraform自动化部署DWS集群快照，包括VPC、子网与安全组创建、DWS集群创建以及快照创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [DWS集群规格（data.huaweicloud_dws_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dws_flavors)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [DWS集群（huaweicloud_dws_cluster）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dws_cluster)
- [DWS快照（huaweicloud_dws_snapshot）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dws_snapshot)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── data.huaweicloud_dws_flavors
        └── huaweicloud_dws_cluster
            └── huaweicloud_dws_snapshot

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_dws_cluster

huaweicloud_networking_secgroup
    └── huaweicloud_dws_cluster
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用区列表

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区列表
variable "availability_zone" {
  description = "The availability zone of the DWS cluster"
  type        = string
  default     = ""
  nullable    = false
}

data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**参数说明**：
- **count**：通过引用输入变量 availability_zone 进行赋值，当未指定可用区时查询当前region下的可用区列表

### 3. 创建虚拟私有云

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = ""
  nullable    = false
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
- **enterprise_project_id**：通过引用输入变量 enterprise_project_id 进行赋值，未指定时使用默认企业项目

### 4. 创建虚拟私有云子网

在TF文件（如main.tf）中添加以下脚本：

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
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**参数说明**：
- **vpc_id**：通过引用虚拟私有云资源的ID进行赋值
- **name**：通过引用输入变量 subnet_name 进行赋值
- **cidr**：通过引用输入变量 subnet_cidr 进行赋值，未指定时基于VPC网段自动划分子网网段
- **gateway_ip**：通过引用输入变量 subnet_gateway_ip 进行赋值，未指定时基于子网网段自动计算网关IP

### 5. 创建安全组

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

variable "security_group_delete_default_rules" {
  description = "Whether to delete the default rules of the security group"
  type        = bool
  default     = true
}

resource "huaweicloud_networking_secgroup" "test" {
  name                  = var.security_group_name
  delete_default_rules  = var.security_group_delete_default_rules
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}
```

**参数说明**：
- **name**：通过引用输入变量 security_group_name 进行赋值
- **delete_default_rules**：通过引用输入变量 security_group_delete_default_rules 进行赋值，用于控制是否删除安全组默认规则
- **enterprise_project_id**：通过引用输入变量 enterprise_project_id 进行赋值，未指定时使用默认企业项目

### 6. 查询DWS集群规格

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询DWS集群规格
variable "cluster_node_type" {
  description = "The flavor of the DWS cluster node"
  type        = string
  default     = ""
  nullable    = false
}

variable "cluster_version" {
  description = "The version of the DWS cluster"
  type        = string
  default     = ""
  nullable    = false
}

variable "cluster_vcpus" {
  description = "The vcpus of the DWS cluster"
  type        = number
  default     = 4
}

variable "cluster_memory" {
  description = "The memory of the DWS cluster"
  type        = number
  default     = 32
}

variable "cluster_datastore_type" {
  description = "The datastore type of the DWS cluster"
  type        = string
  default     = "dws"
}

data "huaweicloud_dws_flavors" "test" {
  count = var.cluster_node_type == "" || var.cluster_version == "" ? 1 : 0

  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  vcpus             = var.cluster_vcpus
  memory            = var.cluster_memory
  datastore_type    = var.cluster_datastore_type
}
```

**参数说明**：
- **count**：通过引用输入变量 cluster_node_type 与 cluster_version 进行赋值，当未指定节点规格或版本时查询DWS集群规格
- **availability_zone**：通过引用输入变量 availability_zone 进行赋值，未指定时使用可用区列表中的第一个可用区
- **vcpus**：通过引用输入变量 cluster_vcpus 进行赋值
- **memory**：通过引用输入变量 cluster_memory 进行赋值
- **datastore_type**：通过引用输入变量 cluster_datastore_type 进行赋值

### 7. 创建DWS集群

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DWS集群
variable "cluster_name" {
  description = "The name of the DWS cluster"
  type        = string
}

variable "cluster_number_of_node" {
  description = "The number of nodes in the DWS cluster"
  type        = number
  default     = 3
}

variable "cluster_number_of_cn" {
  description = "The number of CN nodes in the DWS cluster"
  type        = number
  default     = 3
}

variable "cluster_admin_user_name" {
  description = "The administrator username of the DWS cluster"
  type        = string
}

variable "cluster_admin_user_pwd" {
  description = "The administrator password of the DWS cluster"
  type        = string
  sensitive   = true
}

variable "cluster_volume_type" {
  description = "The volume type of the DWS cluster"
  type        = string
  default     = "SSD"
}

variable "cluster_volume_capacity" {
  description = "The volume capacity of the DWS cluster in GB"
  type        = string
  default     = "100"
}

resource "huaweicloud_dws_cluster" "test" {
  name                  = var.cluster_name
  node_type             = var.cluster_node_type != "" ? var.cluster_node_type : try(data.huaweicloud_dws_flavors.test[0].flavors[0].flavor_id, null)
  number_of_node        = var.cluster_number_of_node
  number_of_cn          = var.cluster_number_of_cn
  version               = var.cluster_version != "" ? var.cluster_version : try(data.huaweicloud_dws_flavors.test[0].flavors[0].datastore_version, null)
  vpc_id                = huaweicloud_vpc.test.id
  network_id            = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  availability_zone     = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  user_name             = var.cluster_admin_user_name
  user_pwd              = var.cluster_admin_user_pwd
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null

  volume {
    type     = var.cluster_volume_type
    capacity = var.cluster_volume_capacity
  }
}
```

**参数说明**：
- **name**：通过引用输入变量 cluster_name 进行赋值
- **node_type**：通过引用输入变量 cluster_node_type 进行赋值，未指定时使用DWS集群规格数据源查询到的规格ID
- **number_of_node**：通过引用输入变量 cluster_number_of_node 进行赋值
- **number_of_cn**：通过引用输入变量 cluster_number_of_cn 进行赋值
- **version**：通过引用输入变量 cluster_version 进行赋值，未指定时使用DWS集群规格数据源查询到的版本
- **vpc_id**：通过引用虚拟私有云资源的ID进行赋值
- **network_id**：通过引用虚拟私有云子网资源的ID进行赋值
- **security_group_id**：通过引用安全组资源的ID进行赋值
- **availability_zone**：通过引用输入变量 availability_zone 进行赋值，未指定时使用可用区列表中的第一个可用区
- **user_name**：通过引用输入变量 cluster_admin_user_name 进行赋值
- **user_pwd**：通过引用输入变量 cluster_admin_user_pwd 进行赋值
- **enterprise_project_id**：通过引用输入变量 enterprise_project_id 进行赋值，未指定时使用默认企业项目
- **volume.type**：通过引用输入变量 cluster_volume_type 进行赋值
- **volume.capacity**：通过引用输入变量 cluster_volume_capacity 进行赋值

### 8. 创建DWS快照

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DWS快照
variable "snapshot_name" {
  description = "The name of the snapshot"
  type        = string
}

variable "snapshot_description" {
  description = "The description of the snapshot"
  type        = string
  default     = ""
}

resource "huaweicloud_dws_snapshot" "test" {
  name        = var.snapshot_name
  cluster_id  = huaweicloud_dws_cluster.test.id
  description = var.snapshot_description
}
```

**参数说明**：
- **name**：通过引用输入变量 snapshot_name 进行赋值
- **cluster_id**：通过引用DWS集群资源的ID进行赋值
- **description**：通过引用输入变量 snapshot_description 进行赋值

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
vpc_name                = "tf_test_dws_vpc"
vpc_cidr                = "192.168.0.0/16"
subnet_name             = "tf_test_dws_subnet"
security_group_name     = "tf_test_dws_sg"
cluster_name            = "tf_test_dws_cluster"
cluster_admin_user_name = "dbadmin"
cluster_admin_user_pwd  = "YourPassword@123"
snapshot_name           = "tf_test_dws_snapshot"
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
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DWS快照
4. 运行 `terraform show` 查看已创建的DWS快照

## 参考信息

- [华为云数据仓库服务产品文档](https://support.huaweicloud.com/dws/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DWS快照最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dws/snapshot)
