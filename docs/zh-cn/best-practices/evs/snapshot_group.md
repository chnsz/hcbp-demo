# 部署磁盘快照组

## 应用场景

云硬盘（Elastic Volume Service，EVS）是华为云提供的高性能、高可靠、可扩展的块存储服务，为ECS实例提供持久化存储。EVS支持多种存储类型，包括SSD、SAS、SATA等，满足不同业务场景的存储需求。

EVS磁盘快照组是EVS服务的重要功能，用于创建多个云硬盘的一致性快照备份。通过磁盘快照组，企业可以确保在特定时间点多个相关云硬盘的数据一致性，这对于数据库集群、分布式应用等需要数据一致性的场景非常重要。本最佳实践将介绍如何使用Terraform自动化部署EVS磁盘快照组，包括ECS实例创建、云硬盘创建、挂载和磁盘快照组配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ECS规格列表查询数据源（data.huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [镜像列表查询数据源（data.huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [ECS实例资源（huaweicloud_compute_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [云硬盘资源（huaweicloud_evs_volume）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evs_volume)
- [云硬盘挂载资源（huaweicloud_compute_volume_attach）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_volume_attach)
- [EVS磁盘快照组资源（huaweicloud_evs_snapshot_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evs_snapshot_group)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones.test
    ├── data.huaweicloud_compute_flavors.test
    ├── huaweicloud_vpc_subnet.test
    └── huaweicloud_compute_instance.test

data.huaweicloud_compute_flavors.test
    ├── data.huaweicloud_images_images.test
    └── huaweicloud_compute_instance.test

data.huaweicloud_images_images.test
    └── huaweicloud_compute_instance.test

huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test

huaweicloud_vpc_subnet.test
    └── huaweicloud_compute_instance.test

huaweicloud_networking_secgroup.test
    └── huaweicloud_compute_instance.test

huaweicloud_compute_instance.test
    └── huaweicloud_compute_volume_attach.test

huaweicloud_evs_volume.test
    └── huaweicloud_compute_volume_attach.test

huaweicloud_compute_volume_attach.test
    └── huaweicloud_evs_snapshot_group.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询可用区信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建ECS实例和云硬盘：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建ECS实例和云硬盘
data "huaweicloud_availability_zones" "test" {}
```

**参数说明**：
- 无需额外参数，数据源会自动获取当前region下的所有可用区信息

### 3. 通过数据源查询ECS规格信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建ECS实例：

```hcl
variable "instance_flavor_id" {
  description = "ECS实例规格ID，如果未指定，将使用匹配条件的第一可用规格"
  type        = string
  default     = ""
}

variable "availability_zone" {
  description = "ECS实例创建可用区，如果未指定，将使用第一可用区"
  type        = string
  default     = ""
}

variable "instance_flavor_performance_type" {
  description = "ECS实例规格性能类型，当instance_flavor_id未指定时用于查询可用规格"
  type        = string
  default     = "normal"
}

variable "instance_flavor_cpu_core_count" {
  description = "ECS实例规格CPU核数，当instance_flavor_id未指定时用于查询可用规格"
  type        = number
  default     = 0
}

variable "instance_flavor_memory_size" {
  description = "ECS实例规格内存大小（GB），当instance_flavor_id未指定时用于查询可用规格"
  type        = number
  default     = 0
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的ECS规格信息，用于创建ECS实例
data "huaweicloud_compute_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
  performance_type  = var.instance_flavor_performance_type
  cpu_core_count    = var.instance_flavor_cpu_core_count
  memory_size       = var.instance_flavor_memory_size
}
```

**参数说明**：
- **availability_zone**：可用区，优先使用输入变量，如果为空则使用可用区数据源的第一个结果
- **performance_type**：性能类型，通过引用输入变量instance_flavor_performance_type进行赋值
- **cpu_core_count**：CPU核数，通过引用输入变量instance_flavor_cpu_core_count进行赋值
- **memory_size**：内存大小，通过引用输入变量instance_flavor_memory_size进行赋值

### 4. 通过数据源查询镜像信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建ECS实例：

```hcl
variable "instance_image_id" {
  description = "ECS实例镜像ID，如果未指定，将使用匹配条件的第一可用镜像"
  type        = string
  default     = ""
}

variable "instance_image_os_type" {
  description = "ECS实例镜像操作系统类型，当instance_image_id未指定时用于查询可用镜像"
  type        = string
  default     = "Ubuntu"
}

variable "instance_image_visibility" {
  description = "ECS实例镜像可见性，当instance_image_id未指定时用于查询可用镜像"
  type        = string
  default     = "public"
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的镜像信息，用于创建ECS实例
data "huaweicloud_images_images" "test" {
  count = var.instance_image_id == "" ? 1 : 0

  flavor_id  = var.instance_flavor_id == "" ? try(data.huaweicloud_compute_flavors.test[0].ids[0], "") : var.instance_flavor_id
  os         = var.instance_image_os_type
  visibility = var.instance_image_visibility
}
```

**参数说明**：
- **flavor_id**：规格ID，优先使用输入变量，如果为空则使用ECS规格数据源的第一个结果
- **os**：操作系统类型，通过引用输入变量instance_image_os_type进行赋值
- **visibility**：可见性，通过引用输入变量instance_image_visibility进行赋值

### 5. 创建VPC

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC资源：

```hcl
variable "vpc_name" {
  description = "VPC名称"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC的CIDR块"
  type        = string
  default     = "192.168.0.0/16"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量vpc_name进行赋值
- **cidr**：VPC的CIDR块，通过引用输入变量vpc_cidr进行赋值

### 6. 创建VPC子网

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC子网资源：

```hcl
variable "subnet_name" {
  description = "VPC子网名称"
  type        = string
}

variable "subnet_cidr" {
  description = "VPC子网的CIDR块，如果未指定，将在现有CIDR地址块内计算子网cidr"
  type        = string
  default     = ""
}

variable "subnet_gateway_ip" {
  description = "VPC子网的网关IP，如果未指定，将在现有CIDR地址块内计算网关IP"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id            = huaweicloud_vpc.test.id
  name              = var.subnet_name
  cidr              = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip        = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
}
```

**参数说明**：
- **vpc_id**：VPC ID，通过引用VPC资源（huaweicloud_vpc.test）的ID进行赋值
- **name**：子网名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网CIDR块，优先使用输入变量，如果为空则通过cidrsubnet函数计算
- **gateway_ip**：网关IP，优先使用输入变量，如果为空则通过cidrhost函数计算
- **availability_zone**：可用区，优先使用输入变量，如果为空则使用可用区数据源的第一个结果

### 7. 创建安全组

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建安全组资源：

```hcl
variable "secgroup_name" {
  description = "安全组名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.secgroup_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组名称，通过引用输入变量secgroup_name进行赋值
- **delete_default_rules**：删除默认规则，设置为true

### 8. 创建ECS实例

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建ECS实例资源：

```hcl
variable "ecs_instance_name" {
  description = "ECS实例名称"
  type        = string
}

variable "key_pair_name" {
  description = "ECS登录密钥对名称"
  type        = string
  default     = ""
}

variable "system_disk_type" {
  description = "系统盘类型"
  type        = string
  default     = "SAS"
}

variable "system_disk_size" {
  description = "系统盘大小（GB）"
  type        = number
  default     = 40
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ECS实例资源
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
- **name**：实例名称，通过引用输入变量ecs_instance_name进行赋值
- **availability_zone**：可用区，优先使用输入变量，如果为空则使用可用区数据源的第一个结果
- **flavor_id**：规格ID，优先使用输入变量，如果为空则使用ECS规格数据源的第一个结果
- **image_id**：镜像ID，优先使用输入变量，如果为空则使用镜像数据源的第一个结果
- **security_groups**：安全组列表，引用安全组资源
- **key_pair**：密钥对名称，通过引用输入变量key_pair_name进行赋值
- **system_disk_type**：系统盘类型，通过引用输入变量system_disk_type进行赋值
- **system_disk_size**：系统盘大小，通过引用输入变量system_disk_size进行赋值

### 9. 创建云硬盘

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建云硬盘资源：

```hcl
variable "volume_configuration" {
  description = "挂载到ECS实例的云硬盘配置列表"
  type = list(object({
    name        = string
    size        = number
    volume_type = string
    device_type = string
  }))
  default = []
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云硬盘资源
resource "huaweicloud_evs_volume" "test" {
  count = length(var.volume_configuration)

  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test.names[0], null) : var.availability_zone
  volume_type       = var.volume_configuration[count.index].volume_type
  name              = var.volume_configuration[count.index].name
  size              = var.volume_configuration[count.index].size
  device_type       = var.volume_configuration[count.index].device_type
}
```

**参数说明**：
- **availability_zone**：可用区，优先使用输入变量，如果为空则使用可用区数据源的第一个结果
- **volume_type**：云硬盘类型，通过引用输入变量volume_configuration中的volume_type进行赋值
- **name**：云硬盘名称，通过引用输入变量volume_configuration中的name进行赋值
- **size**：云硬盘大小，通过引用输入变量volume_configuration中的size进行赋值
- **device_type**：设备类型，通过引用输入变量volume_configuration中的device_type进行赋值

### 10. 挂载云硬盘

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建云硬盘挂载资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云硬盘挂载资源
resource "huaweicloud_compute_volume_attach" "test" {
  count = length(var.volume_configuration)

  instance_id = huaweicloud_compute_instance.test.id
  volume_id   = huaweicloud_evs_volume.test[count.index].id
}
```

**参数说明**：
- **instance_id**：ECS实例ID，通过引用ECS实例资源（huaweicloud_compute_instance.test）的ID进行赋值
- **volume_id**：云硬盘ID，通过引用云硬盘资源（huaweicloud_evs_volume.test）的ID进行赋值

### 11. 创建EVS磁盘快照组

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建EVS磁盘快照组资源：

```hcl
variable "instant_access" {
  description = "是否启用磁盘快照组即时访问"
  type        = bool
  default     = false
}

variable "snapshot_group_name" {
  description = "磁盘快照组名称"
  type        = string
  default     = ""
}

variable "snapshot_group_description" {
  description = "磁盘快照组描述"
  type        = string
  default     = "Created by Terraform"
}

variable "enterprise_project_id" {
  description = "磁盘快照组企业项目ID"
  type        = string
  default     = "0"
}

variable "incremental" {
  description = "是否创建增量快照"
  type        = bool
  default     = false
}

variable "tags" {
  description = "磁盘快照组关联的键值对"
  type        = map(string)
  default = {
    environment = "test"
    created_by  = "terraform"
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建EVS磁盘快照组资源
resource "huaweicloud_evs_snapshot_group" "test" {
  depends_on = [huaweicloud_compute_volume_attach.test]

  server_id             = huaweicloud_compute_instance.test.id
  volume_ids            = length(huaweicloud_evs_volume.test) > 0 ? try([for v in huaweicloud_evs_volume.test : v.id], null) : null
  instant_access        = var.instant_access
  name                  = var.snapshot_group_name
  description           = var.snapshot_group_description
  enterprise_project_id = var.enterprise_project_id
  incremental           = var.incremental
  tags                  = var.tags
}
```

**参数说明**：
- **server_id**：ECS实例ID，通过引用ECS实例资源（huaweicloud_compute_instance.test）的ID进行赋值
- **volume_ids**：云硬盘ID列表，通过引用云硬盘资源列表进行赋值
- **instant_access**：即时访问，通过引用输入变量instant_access进行赋值
- **name**：磁盘快照组名称，通过引用输入变量snapshot_group_name进行赋值
- **description**：磁盘快照组描述，通过引用输入变量snapshot_group_description进行赋值
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值
- **incremental**：增量快照，通过引用输入变量incremental进行赋值
- **tags**：标签，通过引用输入变量tags进行赋值

### 12. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 网络配置
vpc_name    = "evs-test-vpc"
vpc_cidr    = "192.168.0.0/16"
subnet_name = "evs-test-subnet"
secgroup_name = "evs-test-sg"

# ECS实例配置
ecs_instance_name = "evs-test-ecs"

# 云硬盘配置
volume_configuration = [
  {
    name        = "evs-test-volume1"
    size        = 50
    volume_type = "SSD"
    device_type = "VBD"
  },
  {
    name        = "evs-test-volume2"
    size        = 100
    volume_type = "SAS"
    device_type = "SCSI"
  }
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="ecs_instance_name=my-ecs"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 13. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建磁盘快照组
4. 运行 `terraform show` 查看已创建的磁盘快照组

## 参考信息

- [华为云EVS产品文档](https://support.huaweicloud.com/evs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [EVS磁盘快照组最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/evs/snapshot-group)
