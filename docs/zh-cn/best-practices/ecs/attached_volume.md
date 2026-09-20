# 部署绑定磁盘的实例

## 应用场景

弹性云服务器（ECS）是由CPU、内存、操作系统、云硬盘组成的基础计算组件。在实际业务中，系统盘往往仅用于承载操作系统，而数据库、日志、应用数据等需要独立、可扩展的持久化存储空间，此时需要为ECS实例挂载额外的数据盘（云硬盘）。

本最佳实践将介绍如何使用Terraform自动化部署一台绑定数据盘的ECS实例，包括可用区、规格与镜像的自动查询，VPC、子网与安全组的创建，ECS实例的创建，以及云硬盘的创建并挂载至该实例。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ECS规格列表（huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [IMS镜像列表（huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [ECS实例（huaweicloud_compute_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [云硬盘（huaweicloud_evs_volume）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evs_volume)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── data.huaweicloud_compute_flavors
        └── data.huaweicloud_images_images
            └── huaweicloud_compute_instance
                └── huaweicloud_evs_volume

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_compute_instance

huaweicloud_networking_secgroup
    └── huaweicloud_compute_instance
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用区、规格与镜像

在TF文件（如main.tf）中添加以下脚本以查询部署ECS实例所需的可用区、规格与镜像信息：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区列表
variable "availability_zone" {
  description = "The availability zone to which the ECS instance belongs"
  type        = string
  default     = ""
  nullable    = false
}

data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询ECS规格列表
variable "instance_flavor_id" {
  description = "The flavor ID of the ECS instance"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_performance_type" {
  description = "The performance type of the ECS instance flavor"
  type        = string
  default     = "normal"
}

variable "instance_cpu_core_count" {
  description = "The number of CPU cores of the ECS instance"
  type        = number
  default     = 2
}

variable "instance_memory_size" {
  description = "The memory size in GB of the ECS instance"
  type        = number
  default     = 4
}

data "huaweicloud_compute_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  performance_type  = var.instance_performance_type
  cpu_core_count    = var.instance_cpu_core_count
  memory_size       = var.instance_memory_size
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询IMS镜像列表
variable "instance_image_id" {
  description = "The image ID of the ECS instance"
  type        = string
  default     = ""
  nullable    = false
}

variable "instance_image_visibility" {
  description = "The visibility of the ECS instance image"
  type        = string
  default     = "public"
}

variable "instance_image_os" {
  description = "The operating system of the ECS instance image"
  type        = string
  default     = "Ubuntu"
}

data "huaweicloud_images_images" "test" {
  count = var.instance_image_id == "" ? 1 : 0

  flavor_id  = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null)
  visibility = var.instance_image_visibility
  os         = var.instance_image_os
}
```

**参数说明**：
- **count**：通过条件表达式控制数据源是否执行查询，当输入变量已指定对应值时跳过查询
- **availability_zone**：通过引用输入变量 availability_zone 进行赋值，未指定时取可用区列表中的第一个可用区
- **performance_type**：通过引用输入变量 instance_performance_type 进行赋值，用于筛选规格的性能类型
- **cpu_core_count**：通过引用输入变量 instance_cpu_core_count 进行赋值，用于筛选规格的CPU核数
- **memory_size**：通过引用输入变量 instance_memory_size 进行赋值，用于筛选规格的内存大小
- **flavor_id**：通过引用输入变量 instance_flavor_id 进行赋值，未指定时取规格列表中的第一个规格
- **visibility**：通过引用输入变量 instance_image_visibility 进行赋值，用于筛选镜像的可见性
- **os**：通过引用输入变量 instance_image_os 进行赋值，用于筛选镜像的操作系统

### 3. 创建VPC、子网与安全组

在TF文件（如main.tf）中添加以下脚本以创建网络环境：

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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网
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
  description = "The gateway IP address of the subnet"
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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组
variable "security_group_ids" {
  description = "The list of security group IDs of the ECS instance"
  type        = list(string)
  default     = []
  nullable    = false
}

variable "security_group_name" {
  description = "The name of the security group"
  type        = string
  default     = ""

  validation {
    condition     = !(length(var.security_group_ids) < 1 && var.security_group_name == "")
    error_message = "The security group name cannot be empty if the security group ID list is not set"
  }
}

resource "huaweicloud_networking_secgroup" "test" {
  count = length(var.security_group_ids) < 1 ? 1 : 0

  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量 vpc_name 进行赋值
- **cidr**：VPC网段，通过引用输入变量 vpc_cidr 进行赋值
- **vpc_id**：子网所属的VPC ID，引用VPC资源的ID进行赋值
- **cidr**：子网网段，通过引用输入变量 subnet_cidr 进行赋值，未指定时基于VPC网段自动划分子网
- **gateway_ip**：子网网关IP，通过引用输入变量 subnet_gateway_ip 进行赋值，未指定时基于子网网段自动计算
- **count**：通过条件表达式控制安全组是否创建，当已指定安全组ID列表时跳过创建
- **name**：安全组名称，通过引用输入变量 security_group_name 进行赋值
- **delete_default_rules**：是否删除安全组默认规则，设置为 true 以移除默认放通规则

### 4. 创建ECS实例

在TF文件（如main.tf）中添加以下脚本以创建ECS实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ECS实例
variable "instance_name" {
  description = "The name of the ECS instance"
  type        = string
}

variable "instance_admin_password" {
  description = "The login password of the ECS instance"
  type        = string
  sensitive   = true
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = null
}

resource "huaweicloud_compute_instance" "test" {
  name                  = var.instance_name
  image_id              = var.instance_image_id != "" ? var.instance_image_id : try(data.huaweicloud_images_images.test[0].images[0].id, null)
  flavor_id             = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null)
  security_group_ids    = length(var.security_group_ids) > 0 ? var.security_group_ids : huaweicloud_networking_secgroup.test[*].id
  availability_zone     = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  admin_pass            = var.instance_admin_password
  enterprise_project_id = var.enterprise_project_id

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }
}
```

**参数说明**：
- **name**：ECS实例名称，通过引用输入变量 instance_name 进行赋值
- **image_id**：实例镜像ID，通过引用输入变量 instance_image_id 进行赋值，未指定时取镜像列表中的第一个镜像
- **flavor_id**：实例规格ID，通过引用输入变量 instance_flavor_id 进行赋值，未指定时取规格列表中的第一个规格
- **security_group_ids**：实例安全组ID列表，通过引用输入变量 security_group_ids 进行赋值，未指定时使用创建的安全组
- **availability_zone**：实例所属可用区，通过引用输入变量 availability_zone 进行赋值，未指定时取可用区列表中的第一个可用区
- **admin_pass**：实例登录密码，通过引用输入变量 instance_admin_password 进行赋值
- **enterprise_project_id**：实例所属企业项目ID，通过引用输入变量 enterprise_project_id 进行赋值
- **network/uuid**：实例网络所关联的子网ID，引用子网资源的ID进行赋值

### 5. 创建云硬盘并挂载至ECS实例

在TF文件（如main.tf）中添加以下脚本以创建云硬盘并挂载至ECS实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云硬盘并挂载至ECS实例
variable "volume_name" {
  description = "The name of the data volume"
  type        = string
}

variable "volume_type" {
  description = "The type of the data volume"
  type        = string
  default     = "SSD"
}

variable "volume_size" {
  description = "The size of the data volume in GB"
  type        = number
  default     = 10
}

variable "volume_iops" {
  description = "The IOPS(Input/Output Operations Per Second) for the data volume"
  type        = number
  default     = null
}

variable "volume_throughput" {
  description = "The throughput for the data volume"
  type        = number
  default     = null
}

variable "volume_backup_id" {
  description = "The backup ID from which to create the disk"
  type        = string
  default     = null
}

variable "volume_snapshot_id" {
  description = "The snapshot ID from which to create the disk"
  type        = string
  default     = null
}

resource "huaweicloud_evs_volume" "test" {
  server_id             = huaweicloud_compute_instance.test.id
  name                  = var.volume_name
  availability_zone     = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  volume_type           = var.volume_type
  size                  = var.volume_size
  iops                  = var.volume_iops
  throughput            = var.volume_throughput
  backup_id             = var.volume_backup_id
  snapshot_id           = var.volume_snapshot_id
  enterprise_project_id = var.enterprise_project_id
}
```

**参数说明**：
- **server_id**：云硬盘挂载的目标ECS实例ID，引用ECS实例资源的ID进行赋值
- **name**：云硬盘名称，通过引用输入变量 volume_name 进行赋值
- **availability_zone**：云硬盘所属可用区，通过引用输入变量 availability_zone 进行赋值，未指定时取可用区列表中的第一个可用区
- **volume_type**：云硬盘类型，通过引用输入变量 volume_type 进行赋值
- **size**：云硬盘容量（GB），通过引用输入变量 volume_size 进行赋值
- **iops**：云硬盘IOPS，通过引用输入变量 volume_iops 进行赋值，当类型为 GPSSD2 或 ESSD2 时必填
- **throughput**：云硬盘吞吐量，通过引用输入变量 volume_throughput 进行赋值，当类型为 GPSSD2 时必填
- **backup_id**：用于创建云硬盘的备份ID，通过引用输入变量 volume_backup_id 进行赋值
- **snapshot_id**：用于创建云硬盘的快照ID，通过引用输入变量 volume_snapshot_id 进行赋值
- **enterprise_project_id**：云硬盘所属企业项目ID，通过引用输入变量 enterprise_project_id 进行赋值

### 6. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 资源变量
vpc_name                = "tf_test_ecs_instace"
subnet_name             = "tf_test_ecs_instace"
security_group_name     = "tf_test_ecs_instace"
instance_name           = "tf_test_ecs_instace"
instance_admin_password = "YourPassword!"
volume_name             = "tf_test_volume"
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

### 7. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建绑定磁盘的ECS实例
4. 运行 `terraform show` 查看已创建的绑定磁盘的ECS实例

## 参考信息

- [华为云ECS产品文档](https://support.huaweicloud.com/ecs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [ECS绑定磁盘最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ecs/attached-volume)
