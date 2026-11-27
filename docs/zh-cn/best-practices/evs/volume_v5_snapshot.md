# 部署云硬盘与V5快照

## 应用场景

云硬盘（EVS）为弹性云服务器提供持久化块存储，是业务数据落盘的基础。在实际业务中，除了创建云硬盘并挂载至云服务器外，还需要为云硬盘创建快照，以便在数据误删、病毒入侵或系统故障时快速回滚数据。华为云EVS提供V5版本快照能力，支持即时访问、增量快照、标签与元数据等特性，可满足不同数据保护策略的需求。

本最佳实践将介绍如何使用Terraform自动化部署云硬盘与V5快照，包括VPC与子网创建、安全组创建、ECS实例创建、云硬盘创建与元数据配置、云硬盘挂载以及V5快照创建与元数据配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [云服务器规格列表（data.huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [镜像列表（data.huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [弹性云服务器（huaweicloud_compute_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [云硬盘（huaweicloud_evs_volume）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evs_volume)
- [云硬盘元数据（huaweicloud_evs_volume_metadata）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evs_volume_metadata)
- [云硬盘挂载（huaweicloud_compute_volume_attach）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_volume_attach)
- [V5云硬盘快照（huaweicloud_evsv5_snapshot）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evsv5_snapshot)
- [云硬盘快照元数据（huaweicloud_evs_snapshot_metadata）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evs_snapshot_metadata)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    ├── data.huaweicloud_compute_flavors
    │       └── data.huaweicloud_images_images
    ├── huaweicloud_vpc_subnet
    └── huaweicloud_compute_instance
            └── huaweicloud_compute_volume_attach

data.huaweicloud_compute_flavors
    └── huaweicloud_compute_instance

data.huaweicloud_images_images
    └── huaweicloud_compute_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
            └── huaweicloud_compute_instance

huaweicloud_networking_secgroup
    └── huaweicloud_compute_instance

huaweicloud_evs_volume
    ├── huaweicloud_evs_volume_metadata
    ├── huaweicloud_compute_volume_attach
    └── huaweicloud_evsv5_snapshot
            └── huaweicloud_evs_snapshot_metadata
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用区列表

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区列表
data "huaweicloud_availability_zones" "test" {}
```

**参数说明**：
- 该数据源无需额外参数，用于获取当前region下可用的可用区列表，供后续子网、ECS实例和云硬盘选择可用区使用。

### 3. 查询云服务器规格列表

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询云服务器规格列表
variable "instance_flavor_id" {
  description = "The flavor ID of the ECS instance, if not specified, the first available flavor matching the criteria will be used."
  type        = string
  default     = ""
}

variable "availability_zone" {
  description = "The availability zone where the ECS instance will be created, if not specified, the first available zone will be used"
  type        = string
  default     = ""
}

variable "instance_flavor_performance_type" {
  description = "The performance type of the ECS instance flavor"
  type        = string
  default     = "normal"
}

variable "instance_flavor_cpu_core_count" {
  description = "The number of CPU cores for the ECS instance flavor"
  type        = number
  default     = 0
}

variable "instance_flavor_memory_size" {
  description = "The memory size in GB for the ECS instance flavor"
  type        = number
  default     = 0
}

data "huaweicloud_compute_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
  performance_type  = var.instance_flavor_performance_type
  cpu_core_count    = var.instance_flavor_cpu_core_count
  memory_size       = var.instance_flavor_memory_size
}
```

**参数说明**：
- **count**：通过引用输入变量 instance_flavor_id 进行赋值，当未指定规格ID时创建该数据源
- **availability_zone**：通过引用输入变量 availability_zone 进行赋值，未指定时使用可用区列表中的第一个可用区
- **performance_type**：通过引用输入变量 instance_flavor_performance_type 进行赋值
- **cpu_core_count**：通过引用输入变量 instance_flavor_cpu_core_count 进行赋值
- **memory_size**：通过引用输入变量 instance_flavor_memory_size 进行赋值

### 4. 查询镜像列表

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询镜像列表
variable "instance_image_id" {
  description = "The ID of the image used to create the ECS instance, if not specified, the first available image matching the criteria will be used."
  type        = string
  default     = ""
}

variable "instance_image_os_type" {
  description = "The OS type of the ECS instance image"
  type        = string
  default     = "Ubuntu"
}

variable "instance_image_visibility" {
  description = "The visibility of the ECS instance image"
  type        = string
  default     = "public"
}

data "huaweicloud_images_images" "test" {
  count = var.instance_image_id == "" ? 1 : 0

  flavor_id  = var.instance_flavor_id == "" ? try(data.huaweicloud_compute_flavors.test[0].ids[0], "") : var.instance_flavor_id
  os         = var.instance_image_os_type
  visibility = var.instance_image_visibility
}
```

**参数说明**：
- **count**：通过引用输入变量 instance_image_id 进行赋值，当未指定镜像ID时创建该数据源
- **flavor_id**：通过引用输入变量 instance_flavor_id 进行赋值，未指定时使用规格列表中的第一个规格ID
- **os**：通过引用输入变量 instance_image_os_type 进行赋值
- **visibility**：通过引用输入变量 instance_image_visibility 进行赋值

### 5. 创建虚拟私有云

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
  default     = "192.168.0.0/16"
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：通过引用输入变量 vpc_name 进行赋值
- **cidr**：通过引用输入变量 vpc_cidr 进行赋值

### 6. 创建子网

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建子网
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet, if not specified, calculating a subnet cidr within the existing CIDR address block"
  type        = string
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet, if not specified, calculating a gateway IP within the existing CIDR address block"
  type        = string
  default     = ""
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id            = huaweicloud_vpc.test.id
  name              = var.subnet_name
  cidr              = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip        = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
}
```

**参数说明**：
- **vpc_id**：通过引用虚拟私有云资源的ID进行赋值
- **name**：通过引用输入变量 subnet_name 进行赋值
- **cidr**：通过引用输入变量 subnet_cidr 进行赋值，未指定时在VPC网段内自动划分子网网段
- **gateway_ip**：通过引用输入变量 subnet_gateway_ip 进行赋值，未指定时自动计算网关IP
- **availability_zone**：通过引用输入变量 availability_zone 进行赋值，未指定时使用可用区列表中的第一个可用区

### 7. 创建安全组

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组
variable "secgroup_name" {
  description = "The name of the security group"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.secgroup_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：通过引用输入变量 secgroup_name 进行赋值
- **delete_default_rules**：设置为 true，删除安全组默认规则

### 8. 创建弹性云服务器

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建弹性云服务器
variable "ecs_instance_name" {
  description = "The name of the ECS instance"
  type        = string
}

variable "key_pair_name" {
  description = "The name of the key pair for ECS login"
  type        = string
  default     = ""
}

variable "system_disk_type" {
  description = "The type of the system disk"
  type        = string
  default     = "SAS"
}

variable "system_disk_size" {
  description = "The size of the system disk in GB"
  type        = number
  default     = 40
}

resource "huaweicloud_compute_instance" "test" {
  name              = var.ecs_instance_name
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
  flavor_id         = var.instance_flavor_id == "" ? try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, "") : var.instance_flavor_id
  image_id          = var.instance_image_id == "" ? try(data.huaweicloud_images_images.test[0].images[0].id, "") : var.instance_image_id
  security_groups   = [huaweicloud_networking_secgroup.test.name]
  key_pair          = var.key_pair_name
  system_disk_type  = var.system_disk_type
  system_disk_size  = var.system_disk_size

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }
}
```

**参数说明**：
- **name**：通过引用输入变量 ecs_instance_name 进行赋值
- **availability_zone**：通过引用输入变量 availability_zone 进行赋值，未指定时使用可用区列表中的第一个可用区
- **flavor_id**：通过引用输入变量 instance_flavor_id 进行赋值，未指定时使用规格列表中的第一个规格ID
- **image_id**：通过引用输入变量 instance_image_id 进行赋值，未指定时使用镜像列表中的第一个镜像ID
- **security_groups**：通过引用安全组资源的名称进行赋值
- **key_pair**：通过引用输入变量 key_pair_name 进行赋值
- **system_disk_type**：通过引用输入变量 system_disk_type 进行赋值
- **system_disk_size**：通过引用输入变量 system_disk_size 进行赋值
- **network.uuid**：通过引用子网资源的ID进行赋值

### 9. 创建云硬盘

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云硬盘
variable "volume_availability_zone" {
  description = "The availability zone for the volume"
  type        = string
  default     = ""
  nullable    = false
}

variable "volume_name" {
  description = "The name of the volume"
  type        = string
}

variable "volume_type" {
  description = "The type of the volume"
  type        = string
  default     = "SSD"
}

variable "volume_size" {
  description = "The size of the volume"
  type        = number
  default     = 40
}

variable "volume_description" {
  description = "The description of the volume"
  type        = string
  default     = ""
}

variable "volume_multiattach" {
  description = "The volume is shared volume or not"
  type        = bool
  default     = false
}

variable "volume_iops" {
  description = "The IOPS for the volume"
  type        = number
  default     = null
}

variable "volume_throughput" {
  description = "The throughput for the volume"
  type        = number
  default     = null
}

variable "volume_device_type" {
  description = "The device type of disk to create"
  type        = string
  default     = "VBD"
}

variable "enterprise_project_id" {
  description = "The enterprise project ID"
  type        = string
  default     = null
}

variable "volume_tags" {
  description = "The tags of the volume"
  type        = map(string)
  default     = {}
}

variable "charging_mode" {
  description = "The charging mode of the volume"
  type        = string
  default     = "postPaid"
}

variable "period_unit" {
  description = "The period unit of the volume"
  type        = string
  default     = null
}

variable "period" {
  description = "The period of the volume"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "The auto renew of the volume"
  type        = string
  default     = "false"
}

resource "huaweicloud_evs_volume" "test" {
  availability_zone     = var.volume_availability_zone != "" ? var.volume_availability_zone : try(data.huaweicloud_availability_zones.test.names[0], null)
  name                  = var.volume_name
  volume_type           = var.volume_type
  size                  = var.volume_size
  description           = var.volume_description
  multiattach           = var.volume_multiattach
  iops                  = var.volume_iops
  throughput            = var.volume_throughput
  device_type           = var.volume_device_type
  enterprise_project_id = var.enterprise_project_id
  tags                  = var.volume_tags
  charging_mode         = var.charging_mode
  period_unit           = var.period_unit
  period                = var.period
  auto_renew            = var.auto_renew
}
```

**参数说明**：
- **availability_zone**：通过引用输入变量 volume_availability_zone 进行赋值，未指定时使用可用区列表中的第一个可用区
- **name**：通过引用输入变量 volume_name 进行赋值
- **volume_type**：通过引用输入变量 volume_type 进行赋值
- **size**：通过引用输入变量 volume_size 进行赋值
- **description**：通过引用输入变量 volume_description 进行赋值
- **multiattach**：通过引用输入变量 volume_multiattach 进行赋值
- **iops**：通过引用输入变量 volume_iops 进行赋值，仅当云硬盘类型为GPSSD2或ESSD2时有效
- **throughput**：通过引用输入变量 volume_throughput 进行赋值，仅当云硬盘类型为GPSSD2时有效
- **device_type**：通过引用输入变量 volume_device_type 进行赋值
- **enterprise_project_id**：通过引用输入变量 enterprise_project_id 进行赋值
- **tags**：通过引用输入变量 volume_tags 进行赋值
- **charging_mode**：通过引用输入变量 charging_mode 进行赋值
- **period_unit**：通过引用输入变量 period_unit 进行赋值，仅当计费模式为prePaid时有效
- **period**：通过引用输入变量 period 进行赋值，仅当计费模式为prePaid时有效
- **auto_renew**：通过引用输入变量 auto_renew 进行赋值

### 10. 创建云硬盘元数据

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云硬盘元数据
variable "volume_metadata" {
  description = "The volume metadata"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_evs_volume_metadata" "test" {
  volume_id = huaweicloud_evs_volume.test.id
  metadata  = var.volume_metadata
}
```

**参数说明**：
- **volume_id**：通过引用云硬盘资源的ID进行赋值
- **metadata**：通过引用输入变量 volume_metadata 进行赋值

### 11. 挂载云硬盘至弹性云服务器

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下挂载云硬盘至弹性云服务器
resource "huaweicloud_compute_volume_attach" "test" {
  instance_id = huaweicloud_compute_instance.test.id
  volume_id   = huaweicloud_evs_volume.test.id
}
```

**参数说明**：
- **instance_id**：通过引用弹性云服务器资源的ID进行赋值
- **volume_id**：通过引用云硬盘资源的ID进行赋值

### 12. 创建V5云硬盘快照

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建V5云硬盘快照
variable "snapshot_name" {
  description = "The name of the snapshot"
  type        = string
}

variable "snapshot_description" {
  description = "The description of the snapshot"
  type        = string
  default     = ""
}

variable "snapshot_tags" {
  description = "The tags of the snapshot"
  type        = map(string)
  default     = {}
}

variable "snapshot_instant_access" {
  description = "Whether to enable instant access for the snapshot"
  type        = bool
  default     = false
}

variable "snapshot_incremental" {
  description = "Whether to create an incremental snapshot"
  type        = bool
  default     = false
}

resource "huaweicloud_evsv5_snapshot" "test" {
  volume_id             = huaweicloud_evs_volume.test.id
  name                  = var.snapshot_name
  description           = var.snapshot_description
  tags                  = var.snapshot_tags
  enterprise_project_id = var.enterprise_project_id
  instant_access        = var.snapshot_instant_access
  incremental           = var.snapshot_incremental
}
```

**参数说明**：
- **volume_id**：通过引用云硬盘资源的ID进行赋值
- **name**：通过引用输入变量 snapshot_name 进行赋值
- **description**：通过引用输入变量 snapshot_description 进行赋值
- **tags**：通过引用输入变量 snapshot_tags 进行赋值
- **enterprise_project_id**：通过引用输入变量 enterprise_project_id 进行赋值
- **instant_access**：通过引用输入变量 snapshot_instant_access 进行赋值
- **incremental**：通过引用输入变量 snapshot_incremental 进行赋值

### 13. 创建云硬盘快照元数据

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云硬盘快照元数据
variable "snapshot_metadata" {
  description = "The snapshot metadata"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_evs_snapshot_metadata" "test" {
  snapshot_id = huaweicloud_evsv5_snapshot.test.id
  metadata    = var.snapshot_metadata
}
```

**参数说明**：
- **snapshot_id**：通过引用V5云硬盘快照资源的ID进行赋值
- **metadata**：通过引用输入变量 snapshot_metadata 进行赋值

### 14. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证参数
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 网络与云服务器参数
vpc_name          = "tf_test_evs_vpc"
subnet_name       = "tf_test_evs_subnet"
secgroup_name     = "tf_test_evs_sg"
ecs_instance_name = "tf_test_evs_ecs"

# 云硬盘参数
volume_name        = "tf_test_evs_volume"
volume_description = "terraform volume test"

volume_tags = {
  foo = "bar"
  key = "value"
}

volume_metadata = {
  metadata_key = "volume"
  key          = "value"
}

# V5快照参数
snapshot_name        = "tf_test_evs_v5_snapshot"
snapshot_description = "terraform v5 snapshot test"

snapshot_tags = {
  foo = "bar"
  key = "value"
}

snapshot_metadata = {
  metadata_key = "snapshot"
  key          = "value"
}
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

### 15. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建云硬盘与V5快照
4. 运行 `terraform show` 查看已创建的云硬盘与V5快照

## 参考信息

- [华为云EVS产品文档](https://support.huaweicloud.com/evs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [EVS云硬盘与V5快照最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/evs/volume-v5-snapshot)
