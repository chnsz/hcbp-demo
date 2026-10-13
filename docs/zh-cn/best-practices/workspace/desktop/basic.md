# 部署云桌面

## 应用场景

云桌面（Workspace）是华为云提供的桌面虚拟化服务，为企业用户提供安全、便捷的云上办公环境。通过云桌面，企业可以实现数据集中存储、终端轻量化，用户可随时随地通过各种终端设备安全访问云上办公桌面。

本最佳实践将介绍如何使用Terraform自动化部署一台云桌面，包括VPC、子网、安全组、Workspace服务、桌面用户以及云桌面实例的创建，帮助您快速构建云上办公环境。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [云桌面规格列表（data.huaweicloud_workspace_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/workspace_flavors)
- [镜像列表（data.huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)
- [云桌面服务（data.huaweicloud_workspace_service）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/workspace_service)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [云桌面服务（huaweicloud_workspace_service）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_service)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [云桌面用户（huaweicloud_workspace_user）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_user)
- [云桌面（huaweicloud_workspace_desktop）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_desktop)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── data.huaweicloud_workspace_flavors

data.huaweicloud_images_images

data.huaweicloud_workspace_service
    ├── huaweicloud_vpc
    │   └── huaweicloud_vpc_subnet
    ├── huaweicloud_workspace_service
    └── huaweicloud_networking_secgroup
        └── huaweicloud_networking_secgroup_rule

huaweicloud_workspace_user
    └── huaweicloud_workspace_desktop
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用区列表

在TF文件（如main.tf）中添加以下脚本以查询云桌面规格和网络所属的可用区：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询可用区列表
variable "availability_zone" {
  description = "The availability zone to which the cloud desktop flavor and network belong"
  type        = string
  default     = ""
}

data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**参数说明**：
- **count**：当输入变量 availability_zone 为空时创建该数据源，用于自动获取当前region下的可用区列表

### 3. 查询云桌面规格列表

在TF文件（如main.tf）中添加以下脚本以查询满足条件的云桌面规格：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询云桌面规格列表
variable "desktop_flavor_id" {
  description = "The flavor ID of the cloud desktop"
  type        = string
  default     = ""
}

variable "desktop_flavor_os_type" {
  description = "The OS type of the cloud desktop flavor"
  type        = string
  default     = "Windows"
}

variable "desktop_flavor_cpu_core_number" {
  description = "The number of the cloud desktop flavor CPU cores"
  type        = number
  default     = 4
}

variable "desktop_flavor_memory_size" {
  description = "The number of the cloud desktop flavor memories"
  type        = number
  default     = 8
}

data "huaweicloud_workspace_flavors" "test" {
  count = var.desktop_flavor_id == "" ? 1 : 0

  os_type           = var.desktop_flavor_os_type
  vcpus             = var.desktop_flavor_cpu_core_number
  memory            = var.desktop_flavor_memory_size
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
}
```

**参数说明**：
- **count**：当输入变量 desktop_flavor_id 为空时创建该数据源，用于自动查询云桌面规格
- **os_type**：通过引用输入变量 desktop_flavor_os_type 进行赋值，表示云桌面规格的操作系统类型
- **vcpus**：通过引用输入变量 desktop_flavor_cpu_core_number 进行赋值，表示云桌面规格的CPU核数
- **memory**：通过引用输入变量 desktop_flavor_memory_size 进行赋值，表示云桌面规格的内存大小
- **availability_zone**：通过引用输入变量 availability_zone 进行赋值，未指定时默认使用查询到的第一个可用区

### 4. 查询云桌面镜像列表

在TF文件（如main.tf）中添加以下脚本以查询云桌面镜像：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询云桌面镜像列表
variable "desktop_image_id" {
  description = "The specified image ID that the cloud desktop used"
  type        = string
  default     = ""
}

variable "desktop_image_os_type" {
  description = "The OS type of the cloud desktop image"
  type        = string
  default     = "Windows"
}

variable "desktop_image_visibility" {
  description = "The visibility of the cloud desktop image"
  type        = string
  default     = "market"
}

data "huaweicloud_images_images" "test" {
  count = var.desktop_image_id == "" ? 1 : 0

  name_regex = "WORKSPACE"
  os         = var.desktop_image_os_type
  visibility = var.desktop_image_visibility
}
```

**参数说明**：
- **count**：当输入变量 desktop_image_id 为空时创建该数据源，用于自动查询云桌面镜像
- **name_regex**：镜像名称的正则表达式，用于筛选云桌面镜像
- **os**：通过引用输入变量 desktop_image_os_type 进行赋值，表示镜像的操作系统类型
- **visibility**：通过引用输入变量 desktop_image_visibility 进行赋值，表示镜像的可见性

### 5. 查询云桌面服务状态

在TF文件（如main.tf）中添加以下脚本以查询当前region下云桌面服务的开通状态：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询云桌面服务状态
data "huaweicloud_workspace_service" "test" {}
```

**参数说明**：
- 该数据源无需额外参数，用于获取当前region下云桌面服务的状态、VPC ID、网络ID以及安全组信息

### 6. 创建虚拟私有云

在TF文件（如main.tf）中添加以下脚本以创建虚拟私有云：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云
variable "vpc_name" {
  description = "The VPC name"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
}

resource "huaweicloud_vpc" "test" {
  count = data.huaweicloud_workspace_service.test.status == "CLOSED" ? 1 : 0

  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **count**：当云桌面服务未开通时创建该资源
- **name**：通过引用输入变量 vpc_name 进行赋值，表示VPC名称
- **cidr**：通过引用输入变量 vpc_cidr 进行赋值，表示VPC的网段

### 7. 创建虚拟私有云子网

在TF文件（如main.tf）中添加以下脚本以创建虚拟私有云子网：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云子网
variable "subnet_name" {
  description = "The subnet name"
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

resource "huaweicloud_vpc_subnet" "test" {
  count = data.huaweicloud_workspace_service.test.status == "CLOSED" ? 1 : 0

  vpc_id     = try(huaweicloud_vpc.test[0].id, null)
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(try(huaweicloud_vpc.test[0].cidr, "192.168.0.0/16"), 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(try(huaweicloud_vpc.test[0].cidr, "192.168.0.0/16"), 8, 0), 1) : var.subnet_gateway_ip
}
```

**参数说明**：
- **count**：当云桌面服务未开通时创建该资源
- **vpc_id**：通过引用上一步创建的VPC ID进行赋值
- **name**：通过引用输入变量 subnet_name 进行赋值，表示子网名称
- **cidr**：通过引用输入变量 subnet_cidr 进行赋值，未指定时基于VPC网段自动划分子网网段
- **gateway_ip**：通过引用输入变量 subnet_gateway_ip 进行赋值，未指定时基于子网网段自动计算网关IP

### 8. 开通云桌面服务

在TF文件（如main.tf）中添加以下脚本以开通云桌面服务：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下开通云桌面服务
resource "huaweicloud_workspace_service" "test" {
  count = data.huaweicloud_workspace_service.test.status == "CLOSED" ? 1 : 0

  access_mode = "INTERNET"
  vpc_id      = try(huaweicloud_vpc.test[0].id, null)
  network_ids = [
    try(huaweicloud_vpc_subnet.test[0].id, null),
  ]
}
```

**参数说明**：
- **count**：当云桌面服务未开通时创建该资源
- **access_mode**：云桌面服务的接入方式，此处设置为 `INTERNET` 表示通过互联网接入
- **vpc_id**：通过引用上一步创建的VPC ID进行赋值
- **network_ids**：通过引用上一步创建的子网ID进行赋值，表示云桌面服务使用的网络

### 9. 创建安全组

在TF文件（如main.tf）中添加以下脚本以创建安全组：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组
variable "security_group_name" {
  description = "The security group name"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  count = data.huaweicloud_workspace_service.test.status == "CLOSED" ? 1 : 0

  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **count**：当云桌面服务未开通时创建该资源
- **name**：通过引用输入变量 security_group_name 进行赋值，表示安全组名称
- **delete_default_rules**：设置为 `true` 表示删除安全组的默认规则

### 10. 创建安全组规则

在TF文件（如main.tf）中添加以下脚本以创建安全组规则：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组规则
resource "huaweicloud_networking_secgroup_rule" "test" {
  count = data.huaweicloud_workspace_service.test.status == "CLOSED" ? 1 : 0

  security_group_id = try(huaweicloud_networking_secgroup.test[0].id, null)
  direction         = "egress"
  ethertype         = "IPv4"
  remote_ip_prefix  = "0.0.0.0/0"
  priority          = 1
}
```

**参数说明**：
- **count**：当云桌面服务未开通时创建该资源
- **security_group_id**：通过引用上一步创建的安全组ID进行赋值
- **direction**：规则方向，此处设置为 `egress` 表示出方向
- **ethertype**：IP协议类型，此处设置为 `IPv4`
- **remote_ip_prefix**：远端IP地址范围，此处设置为 `0.0.0.0/0` 表示允许所有地址
- **priority**：规则的优先级，数值越小优先级越高

### 11. 创建云桌面用户

在TF文件（如main.tf）中添加以下脚本以创建云桌面用户：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云桌面用户
variable "desktop_user_name" {
  description = "The user name that the cloud desktop used"
  type        = string
}

variable "desktop_user_email" {
  description = "The email address that the user used"
  type        = string
}

resource "huaweicloud_workspace_user" "test" {
  depends_on = [huaweicloud_workspace_service.test]

  name  = var.desktop_user_name
  email = var.desktop_user_email

  account_expires            = "0"
  password_never_expires     = false
  enable_change_password     = true
  next_login_change_password = true
  disabled                   = false
}
```

**参数说明**：
- **depends_on**：显式依赖云桌面服务，确保服务开通后再创建用户
- **name**：通过引用输入变量 desktop_user_name 进行赋值，表示用户名称
- **email**：通过引用输入变量 desktop_user_email 进行赋值，表示用户邮箱
- **account_expires**：账号过期时间，设置为 `0` 表示永不过期
- **password_never_expires**：密码是否永不过期，此处设置为 `false`
- **enable_change_password**：是否允许修改密码，此处设置为 `true`
- **next_login_change_password**：下次登录是否需要修改密码，此处设置为 `true`
- **disabled**：是否禁用用户，此处设置为 `false`

### 12. 创建云桌面

在TF文件（如main.tf）中添加以下脚本以创建云桌面：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云桌面
variable "cloud_desktop_name" {
  description = "The cloud desktop name"
  type        = string
}

variable "desktop_user_group_name" {
  description = "The name of the user group that cloud desktop used"
  type        = string
  default     = "users"
}

variable "desktop_root_volume_type" {
  description = "The storage type of system disk"
  type        = string
  default     = "SSD"
}

variable "desktop_root_volume_size" {
  description = "The storage capacity of system disk"
  type        = number
  default     = 100
}

variable "desktop_data_volumes" {
  description = "The storage configuration of data disks"
  type        = list(object({
    type = string
    size = number
  }))
  default     = [
    {
      type = "SSD",
      size = 100,
    },
  ]
}

resource "huaweicloud_workspace_desktop" "test" {
  depends_on = [huaweicloud_workspace_user.test]

  flavor_id         = var.desktop_flavor_id == "" ? try([for o in data.huaweicloud_workspace_flavors.test[0].flavors : o.id if !strcontains(lower(o.description), "flexus")][0], null) : var.desktop_flavor_id
  image_type        = var.desktop_image_visibility
  image_id          = var.desktop_image_id == "" ? try(data.huaweicloud_images_images.test[0].images[0].id, null) : var.desktop_image_id
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
  vpc_id            = data.huaweicloud_workspace_service.test.status != "CLOSED" ? data.huaweicloud_workspace_service.test.vpc_id : try(huaweicloud_vpc.test[0].id, null)
  security_groups   = data.huaweicloud_workspace_service.test.status != "CLOSED" ? concat(
    data.huaweicloud_workspace_service.test.desktop_security_group[*].id,
    data.huaweicloud_workspace_service.test.infrastructure_security_group[*].id,
    try(huaweicloud_networking_secgroup.test[0].id, []),
  ) : concat(
    try(huaweicloud_workspace_service.test[0].desktop_security_group[*].id, []),
    try(huaweicloud_workspace_service.test[0].infrastructure_security_group[*].id, []),
    try(huaweicloud_networking_secgroup.test[0].id, []),
  )

  dynamic "nic" {
    for_each = data.huaweicloud_workspace_service.test.status != "CLOSED" ? data.huaweicloud_workspace_service.test.network_ids : try([huaweicloud_vpc_subnet.test[0].id], [])

    content {
      network_id = nic.value
    }
  }

  name       = var.cloud_desktop_name
  user_name  = huaweicloud_workspace_user.test.name
  user_email = huaweicloud_workspace_user.test.email
  user_group = var.desktop_user_group_name

  root_volume {
    type = var.desktop_root_volume_type
    size = var.desktop_root_volume_size
  }

  dynamic "data_volume" {
    for_each = var.desktop_data_volumes

    content {
      type = data_volume.value["type"]
      size = data_volume.value["size"]
    }
  }

  lifecycle {
    ignore_changes = [
      flavor_id,
      image_id,
      availability_zone,
    ]
  }
}
```

**参数说明**：
- **depends_on**：显式依赖云桌面用户，确保用户创建后再创建云桌面
- **flavor_id**：通过引用输入变量 desktop_flavor_id 进行赋值，未指定时自动从规格列表中筛选非Flexus规格
- **image_type**：通过引用输入变量 desktop_image_visibility 进行赋值，表示镜像类型
- **image_id**：通过引用输入变量 desktop_image_id 进行赋值，未指定时自动使用查询到的第一个镜像
- **availability_zone**：通过引用输入变量 availability_zone 进行赋值，未指定时默认使用查询到的第一个可用区
- **vpc_id**：云桌面服务已开通时使用服务关联的VPC，否则使用创建的VPC
- **security_groups**：云桌面使用的安全组列表，包含云桌面安全组、基础设施安全组以及创建的安全组
- **nic**：云桌面的网卡配置，通过动态块引用云桌面服务的网络ID或创建的子网ID
- **name**：通过引用输入变量 cloud_desktop_name 进行赋值，表示云桌面名称
- **user_name**：通过引用云桌面用户的名称进行赋值
- **user_email**：通过引用云桌面用户的邮箱进行赋值
- **user_group**：通过引用输入变量 desktop_user_group_name 进行赋值，表示用户所属用户组
- **root_volume**：系统盘配置，**type** 通过引用输入变量 desktop_root_volume_type 进行赋值，**size** 通过引用输入变量 desktop_root_volume_size 进行赋值
- **data_volume**：数据盘配置，通过动态块引用输入变量 desktop_data_volumes 进行赋值
- **lifecycle**：忽略 flavor_id、image_id、availability_zone 的变更，避免因自动查询结果变化导致资源重建

### 13. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 资源变量
vpc_name            = "tf_test_vpc"
vpc_cidr            = "192.168.0.0/16"
subnet_name         = "tf_test_subnet"
security_group_name = "tf_test_security_group"
desktop_user_name   = "tf_test_user"
desktop_user_email  = "test@example.com"
cloud_desktop_name  = "tf-test-desktop"
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

### 14. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建云桌面
4. 运行 `terraform show` 查看已创建的云桌面

## 参考信息

- [华为云Workspace产品文档](https://support.huaweicloud.com/workspace/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Workspace云桌面最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/workspace/desktop/basic)
