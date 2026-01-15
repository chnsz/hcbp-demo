# 部署跨账号迁移数据镜像

## 应用场景

镜像服务（Image Management Service，IMS）是华为云提供的镜像管理服务，支持镜像的创建、共享、复制等功能。通过跨账号迁移数据镜像，可以将一个账号中的数据盘镜像共享给另一个账号，实现跨账号的数据迁移和镜像共享。本最佳实践将介绍如何使用Terraform自动化部署跨账号迁移数据镜像，包括在共享账号中创建ECS实例、数据盘和数据镜像，将镜像共享给接受账号，在接受账号中接受共享镜像并使用共享镜像创建数据盘。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区数据源（huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ECS规格数据源（huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [镜像数据源（huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)
- [项目数据源（huaweicloud_identity_auth_projects）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/identity_auth_projects)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [ECS实例资源（huaweicloud_compute_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [云硬盘资源（huaweicloud_evs_volume）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evs_volume)
- [数据盘镜像资源（huaweicloud_ims_evs_data_image）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ims_evs_data_image)
- [镜像共享资源（huaweicloud_images_image_share）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/images_image_share)
- [镜像共享接受资源（huaweicloud_images_image_share_accepter）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/images_image_share_accepter)
- [云硬盘V3资源（huaweicloud_evsv3_volume）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/evsv3_volume)
- [云硬盘挂载资源（huaweicloud_compute_volume_attach）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_volume_attach)

### 资源/数据源依赖关系

```text
data.huaweicloud_availability_zones
    ├── data.huaweicloud_compute_flavors
    │   └── data.huaweicloud_images_images
    │       └── huaweicloud_compute_instance
    └── huaweicloud_evs_volume
        └── huaweicloud_ims_evs_data_image
            └── huaweicloud_images_image_share
                └── huaweicloud_images_image_share_accepter
                    └── huaweicloud_evsv3_volume
                        └── huaweicloud_compute_volume_attach

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_compute_instance

huaweicloud_networking_secgroup
    └── huaweicloud_compute_instance

data.huaweicloud_identity_auth_projects
    └── huaweicloud_images_image_share
```

> 注意：本最佳实践涉及两个账号：共享账号（sharer）和接受账号（accepter）。需要在Terraform配置中配置两个provider，分别对应两个账号的鉴权信息。镜像共享后，接受账号需要接受共享才能使用共享镜像。

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

> 注意：本最佳实践需要配置两个provider，分别对应共享账号和接受账号。在provider配置中需要分别指定两个账号的access_key和secret_key。

### 2. 查询共享账号数据源

在TF文件（如main.tf）中添加以下脚本以查询共享账号的可用区、ECS规格和镜像信息：

```hcl
variable "region_name" {
  description = "The region where resources will be created"
  type        = string
}

variable "instance_flavor_id" {
  description = "The ID of the ECS instance flavor"
  type        = string
  default     = ""
  nullable    = true
}

variable "instance_flavor_performance_type" {
  description = "The performance type of the ECS instance flavor"
  type        = string
  default     = "normal"
}

variable "instance_flavor_cpu_core_count" {
  description = "The CPU core count of the ECS instance flavor"
  type        = number
  default     = 2
}

variable "instance_flavor_memory_size" {
  description = "The memory size of the ECS instance flavor (GB)"
  type        = number
  default     = 4
}

variable "instance_image_id" {
  description = "The ID of the ECS instance image"
  type        = string
  default     = ""
  nullable    = true
}

variable "instance_image_visibility" {
  description = "The visibility of the ECS instance image"
  type        = string
  default     = "public"
}

variable "instance_image_os" {
  description = "The OS of the ECS instance image"
  type        = string
  default     = "Ubuntu"
}

# 获取指定region下共享账号的可用区信息，用于创建ECS实例和数据盘
data "huaweicloud_availability_zones" "test" {
  provider = huaweicloud.sharer
}

# 获取指定region下共享账号的ECS规格信息，用于创建ECS实例
data "huaweicloud_compute_flavors" "test" {
  count = var.instance_flavor_id == "" ? 1 : 0

  provider = huaweicloud.sharer

  availability_zone = try(data.huaweicloud_availability_zones.test.names[0], null)
  performance_type  = var.instance_flavor_performance_type
  cpu_core_count    = var.instance_flavor_cpu_core_count
  memory_size       = var.instance_flavor_memory_size
}

# 获取指定region下共享账号的镜像信息，用于创建ECS实例
data "huaweicloud_images_images" "test" {
  count = var.instance_image_id == "" ? 1 : 0

  provider = huaweicloud.sharer

  flavor_id  = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null)
  visibility = var.instance_image_visibility
  os         = var.instance_image_os
}
```

**参数说明**：
- **provider**：指定使用共享账号的provider（huaweicloud.sharer）
- 其他参数说明与常规ECS实例创建相同

### 3. 创建共享账号网络资源

在TF文件（如main.tf）中添加以下脚本以创建共享账号的VPC、子网和安全组：

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

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = null
}

variable "subnet_name" {
  description = "The name of the VPC subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the VPC subnet"
  type        = string
  default     = ""
  nullable    = true
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the VPC subnet"
  type        = string
  default     = ""
  nullable    = true
}

variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

# 在指定region下创建共享账号的VPC资源
resource "huaweicloud_vpc" "test" {
  provider = huaweicloud.sharer

  name                  = var.vpc_name
  cidr                  = var.vpc_cidr
  enterprise_project_id = var.enterprise_project_id
}

# 在指定region下创建共享账号的VPC子网资源
resource "huaweicloud_vpc_subnet" "test" {
  provider = huaweicloud.sharer

  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}

# 在指定region下创建共享账号的安全组资源
resource "huaweicloud_networking_secgroup" "test" {
  provider = huaweicloud.sharer

  name                  = var.security_group_name
  delete_default_rules  = true
  enterprise_project_id = var.enterprise_project_id
}
```

**参数说明**：
- **provider**：指定使用共享账号的provider（huaweicloud.sharer）
- 其他参数说明与常规VPC、子网、安全组创建相同

### 4. 创建共享账号ECS实例和数据盘

在TF文件（如main.tf）中添加以下脚本以创建共享账号的ECS实例和数据盘：

```hcl
variable "instance_name" {
  description = "The name of the ECS instance to be created"
  type        = string
}

variable "administrator_password" {
  description = "The password of the administrator for the ECS instance"
  type        = string
  sensitive   = true
}

variable "data_volume_name" {
  description = "The name of the data volume to be created and attached to ECS instance"
  type        = string
}

variable "data_volume_type" {
  description = "The type of the data volume"
  type        = string
  default     = "SAS"
}

variable "data_volume_size" {
  description = "The size of the data volume in GB"
  type        = number
  default     = 10
}

# 在指定region下创建共享账号的ECS实例资源
resource "huaweicloud_compute_instance" "test" {
  provider = huaweicloud.sharer

  name                        = var.instance_name
  availability_zone           = try(data.huaweicloud_availability_zones.test.names[0], null)
  flavor_id                   = var.instance_flavor_id != "" ? var.instance_flavor_id : try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null)
  image_id                    = var.instance_image_id != "" ? var.instance_image_id : try(data.huaweicloud_images_images.test[0].images[0].id, null)
  security_group_ids          = [huaweicloud_networking_secgroup.test.id]
  admin_pass                  = var.administrator_password
  delete_disks_on_termination = true
  enterprise_project_id       = var.enterprise_project_id

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }

  lifecycle {
    ignore_changes = [
      availability_zone,
      flavor_id,
      image_id,
      admin_pass,
    ]
  }
}

# 在指定region下创建共享账号的数据盘资源
resource "huaweicloud_evs_volume" "test" {
  provider = huaweicloud.sharer

  name              = var.data_volume_name
  server_id         = huaweicloud_compute_instance.test.id
  availability_zone = try(data.huaweicloud_availability_zones.test.names[0], null)
  volume_type       = var.data_volume_type
  size              = var.data_volume_size
}
```

**参数说明**：
- **provider**：指定使用共享账号的provider（huaweicloud.sharer）
- **server_id**：数据盘挂载的ECS实例ID，通过引用ECS实例资源进行赋值
- 其他参数说明与常规ECS实例和数据盘创建相同

### 5. 创建数据盘镜像

在TF文件（如main.tf）中添加以下脚本以从数据盘创建数据镜像：

```hcl
variable "data_image_name" {
  description = "The name of the data disk image to be created"
  type        = string
}

variable "data_image_description" {
  description = "The description of the data disk image"
  type        = string
  default     = ""
}

# 在指定region下创建共享账号的数据盘镜像资源
resource "huaweicloud_ims_evs_data_image" "test" {
  provider = huaweicloud.sharer

  name                  = var.data_image_name
  volume_id             = huaweicloud_evs_volume.test.id
  description           = var.data_image_description
  enterprise_project_id = var.enterprise_project_id
}
```

**参数说明**：
- **provider**：指定使用共享账号的provider（huaweicloud.sharer）
- **name**：数据镜像名称，通过引用输入变量data_image_name进行赋值
- **volume_id**：数据盘ID，通过引用数据盘资源进行赋值
- **description**：数据镜像描述，通过引用输入变量data_image_description进行赋值，可选参数
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值，可选参数

> 注意：数据盘镜像创建需要从已存在的数据盘创建，确保数据盘已创建并包含需要迁移的数据。

### 6. 共享镜像给接受账号

在TF文件（如main.tf）中添加以下脚本以将镜像共享给接受账号：

```hcl
# 获取指定region下接受账号的项目信息，用于镜像共享
data "huaweicloud_identity_auth_projects" "test" {
  provider = huaweicloud.accepter
}

locals {
  accepter_project_ids = [for project in data.huaweicloud_identity_auth_projects.test.projects : project.id if project.name == var.region_name]
}

# 在指定region下创建共享账号的镜像共享资源
resource "huaweicloud_images_image_share" "test" {
  provider = huaweicloud.sharer

  source_image_id    = huaweicloud_ims_evs_data_image.test.id
  target_project_ids = local.accepter_project_ids
}
```

**参数说明**：
- **provider**：指定使用共享账号的provider（huaweicloud.sharer）
- **source_image_id**：源镜像ID，通过引用数据盘镜像资源进行赋值
- **target_project_ids**：目标项目ID列表，通过查询接受账号的项目信息进行赋值

> 注意：镜像共享需要指定接受账号的项目ID。通过查询接受账号的项目信息，可以获取对应区域的项目ID。

### 7. 接受账号接受共享镜像

在TF文件（如main.tf）中添加以下脚本以在接受账号中接受共享镜像：

```hcl
# 在指定region下创建接受账号的镜像共享接受资源
resource "huaweicloud_images_image_share_accepter" "test" {
  provider = huaweicloud.accepter

  image_id = huaweicloud_ims_evs_data_image.test.id

  depends_on = [huaweicloud_images_image_share.test]
}
```

**参数说明**：
- **provider**：指定使用接受账号的provider（huaweicloud.accepter）
- **image_id**：共享镜像ID，通过引用数据盘镜像资源进行赋值
- **depends_on**：显式依赖关系，确保在镜像共享创建后再接受共享

> 注意：接受账号需要接受共享镜像后才能使用。接受共享镜像后，可以在接受账号中使用共享镜像创建数据盘。

### 8. 创建接受账号资源（可选）

在TF文件（如main.tf）中添加以下脚本以在接受账号中创建ECS实例和使用共享镜像创建数据盘：

```hcl
variable "accepter_vpc_name" {
  description = "The name of the VPC in accepter account"
  type        = string
}

variable "accepter_vpc_cidr" {
  description = "The CIDR block of the VPC in accepter account"
  type        = string
  default     = "192.168.0.0/16"
}

variable "accepter_subnet_name" {
  description = "The name of the VPC subnet in accepter account"
  type        = string
}

variable "accepter_security_group_name" {
  description = "The name of the security group in accepter account"
  type        = string
}

variable "accepter_instance_flavor_id" {
  description = "The ID of the ECS instance flavor in accepter account"
  type        = string
  default     = ""
  nullable    = true
}

variable "accepter_instance_image_id" {
  description = "The ID of the ECS instance image in accepter account"
  type        = string
  default     = ""
  nullable    = true
}

variable "accepter_instance_name" {
  description = "The name of the ECS instance to be created in accepter account (optional, for creating new instance)"
  type        = string
}

variable "accepter_data_volume_name" {
  description = "The name of the data volume to be created from shared image in accepter account"
  type        = string
}

variable "accepter_data_volume_type" {
  description = "The type of the data volume in accepter account"
  type        = string
  default     = "SAS"
}

variable "accepter_data_volume_size" {
  description = "The size of the data volume in accepter account in GB"
  type        = number
  default     = 20
}

# 获取指定region下接受账号的可用区信息
data "huaweicloud_availability_zones" "accepter" {
  provider = huaweicloud.accepter
}

# 在指定region下创建接受账号的VPC资源
resource "huaweicloud_vpc" "accepter" {
  provider = huaweicloud.accepter

  name = var.accepter_vpc_name
  cidr = var.accepter_vpc_cidr
}

# 在指定region下创建接受账号的VPC子网资源
resource "huaweicloud_vpc_subnet" "accepter" {
  provider = huaweicloud.accepter

  vpc_id     = huaweicloud_vpc.accepter.id
  name       = var.accepter_subnet_name
  cidr       = cidrsubnet(huaweicloud_vpc.accepter.cidr, 8, 0)
  gateway_ip = cidrhost(cidrsubnet(huaweicloud_vpc.accepter.cidr, 8, 0), 1)
}

# 在指定region下创建接受账号的安全组资源
resource "huaweicloud_networking_secgroup" "accepter" {
  provider = huaweicloud.accepter

  name                 = var.accepter_security_group_name
  delete_default_rules = true
}

# 获取指定region下接受账号的ECS规格信息
data "huaweicloud_compute_flavors" "accepter" {
  count = var.accepter_instance_flavor_id == "" ? 1 : 0

  provider = huaweicloud.accepter

  availability_zone = try(data.huaweicloud_availability_zones.accepter.names[0], null)
  performance_type  = var.instance_flavor_performance_type
  cpu_core_count    = var.instance_flavor_cpu_core_count
  memory_size       = var.instance_flavor_memory_size
}

# 获取指定region下接受账号的镜像信息
data "huaweicloud_images_images" "accepter" {
  count = var.accepter_instance_image_id == "" ? 1 : 0

  provider = huaweicloud.accepter

  flavor_id  = var.accepter_instance_flavor_id != "" ? var.accepter_instance_flavor_id : try(data.huaweicloud_compute_flavors.accepter[0].flavors[0].id, null)
  visibility = var.instance_image_visibility
  os         = var.instance_image_os
}

# 在指定region下创建接受账号的ECS实例资源
resource "huaweicloud_compute_instance" "accepter" {
  provider = huaweicloud.accepter

  name                        = var.accepter_instance_name
  availability_zone           = try(data.huaweicloud_availability_zones.accepter.names[0], null)
  flavor_id                   = var.accepter_instance_flavor_id != "" ? var.accepter_instance_flavor_id : try(data.huaweicloud_compute_flavors.accepter[0].flavors[0].id, null)
  image_id                    = var.accepter_instance_image_id != "" ? var.accepter_instance_image_id : try(data.huaweicloud_images_images.accepter[0].images[0].id, null)
  security_group_ids          = [huaweicloud_networking_secgroup.accepter.id]
  admin_pass                  = var.administrator_password
  delete_disks_on_termination = true

  network {
    uuid = huaweicloud_vpc_subnet.accepter.id
  }

  depends_on = [huaweicloud_images_image_share_accepter.test]

  lifecycle {
    ignore_changes = [
      availability_zone,
      flavor_id,
      image_id,
      admin_pass,
    ]
  }
}

# 在指定region下创建接受账号的数据盘资源（使用共享镜像）
resource "huaweicloud_evsv3_volume" "accepter" {
  provider = huaweicloud.accepter

  name              = var.accepter_data_volume_name
  availability_zone = try(data.huaweicloud_availability_zones.accepter.names[0], null)
  volume_type       = var.accepter_data_volume_type
  size              = var.accepter_data_volume_size
  image_id          = huaweicloud_images_image_share_accepter.test.image_id
}

# 在指定region下创建接受账号的云硬盘挂载资源
resource "huaweicloud_compute_volume_attach" "accepter" {
  provider = huaweicloud.accepter

  instance_id = huaweicloud_compute_instance.accepter.id
  volume_id   = huaweicloud_evsv3_volume.accepter.id
}
```

**参数说明**：
- **provider**：指定使用接受账号的provider（huaweicloud.accepter）
- **image_id**：数据盘镜像ID，通过引用镜像共享接受资源进行赋值，使用共享镜像创建数据盘
- **depends_on**：显式依赖关系，确保在接受共享镜像后再创建ECS实例和数据盘
- 其他参数说明与常规ECS实例和数据盘创建相同

> 注意：接受账号可以使用共享镜像创建数据盘。创建数据盘时，通过指定image_id参数使用共享镜像。数据盘创建后，可以挂载到ECS实例使用。

### 9. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"

# 共享账号变量
access_key = "your_sharer_access_key"
secret_key = "your_sharer_secret_key"

# 接受账号变量
accepter_access_key = "your_accepter_access_key"
accepter_secret_key = "your_accepter_secret_key"

# 共享账号资源配置
vpc_name                     = "tf_test_whole_image_vpc"
subnet_name                  = "tf_test_whole_image_subnet"
security_group_name          = "tf_test_whole_image_sg"
instance_name                = "tf_test_whole_image_ecs"
administrator_password       = "YourPassword@12!"
data_volume_name             = "tf_test_data_volume"
data_image_name              = "tf_test_data_image"

# 接受账号资源配置
accepter_vpc_name            = "tf_test_accepter_vpc"
accepter_subnet_name         = "tf_test_accepter_subnet"
accepter_security_group_name = "tf_test_accepter_sg"
accepter_instance_name       = "tf_test_accepter_instance"
accepter_data_volume_name    = "tf_test_accepter_data_volume"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值，特别是：
   - `region_name`需要设置为资源所在的区域
   - `access_key`和`secret_key`需要设置为共享账号的鉴权信息
   - `accepter_access_key`和`accepter_secret_key`需要设置为接受账号的鉴权信息
   - 共享账号和接受账号的资源名称、网络配置等参数需要根据实际需求设置
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="region_name=cn-north-4" -var="vpc_name=my-vpc"`
2. 环境变量：`export TF_VAR_region_name=cn-north-4` 和 `export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。本最佳实践需要配置两个账号的鉴权信息，请确保两个账号的access_key和secret_key正确配置。

### 10. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建跨账号迁移数据镜像：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建共享账号和接受账号的资源
4. 运行 `terraform show` 查看已创建的跨账号迁移数据镜像详情

> 注意：跨账号迁移数据镜像需要两个账号的鉴权信息，请确保两个账号的provider配置正确。镜像共享后，接受账号需要接受共享才能使用共享镜像。数据盘镜像创建需要从已存在的数据盘创建，确保数据盘已创建并包含需要迁移的数据。

## 参考信息

- [华为云IMS产品文档](https://support.huaweicloud.com/ims/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [跨账号迁移数据镜像最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ims/cross-account-migration-with-data-image)
