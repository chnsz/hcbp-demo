# 部署按需计费的云桌面

## 应用场景

华为云云桌面（Workspace）是一种基于云计算的桌面虚拟化服务，为企业用户提供安全、便捷的云上办公解决方案。
云桌面提供了远程桌面访问能力，使用户可以通过各种终端设备随时随地访问自己的云上办公环境，同时集中管理数据和应用，提高安全性和工作效率。
按需计费模式让企业能够根据实际使用量灵活付费，无需预付大量资金，适合临时项目或用量波动的场景。
本最佳实践将介绍如何使用Terraform自动化部署按需计费的云桌面实例。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [云桌面规格列表查询数据源（data.huaweicloud_workspace_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/workspace_flavors)
- [IMS镜像列表查询数据源（data.huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)
- [云桌面服务查询数据源（data.huaweicloud_workspace_service）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/workspace_service)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [云桌面服务资源（huaweicloud_workspace_service）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_service)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则资源（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [云桌面用户资源（huaweicloud_workspace_user）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_user)
- [云桌面实例资源（huaweicloud_workspace_desktop）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_desktop)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── data.huaweicloud_workspace_flavors
        └── huaweicloud_workspace_desktop

data.huaweicloud_images_images
    └── huaweicloud_workspace_desktop

data.huaweicloud_workspace_service
    ├── huaweicloud_vpc
    ├── huaweicloud_vpc_subnet
    ├── huaweicloud_workspace_service
    ├── huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule
    └── huaweicloud_workspace_desktop

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_workspace_service

huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule
    └── huaweicloud_workspace_desktop

huaweicloud_workspace_service
    └── huaweicloud_workspace_desktop

huaweicloud_workspace_user
    └── huaweicloud_workspace_desktop
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询云桌面实例资源创建所需的可用区（data.huaweicloud_availability_zones）

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建云桌面实例：

```hcl
variable "availability_zone" {
  description = "The availability zone to which the cloud desktop flavor and network belong"
  type        = string
  default     = ""
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建云桌面实例
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行可用区列表查询数据源，仅当 `var.availability_zone` 为空时创建数据源（即执行可用区列表查询）

### 3. 通过数据源查询云桌面实例资源创建所需的规格（data.huaweicloud_workspace_flavors）

在TF文件中添加以下脚本以告知Terraform查询符合条件的云桌面规格：

```hcl
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

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的云桌面规格信息，用于创建云桌面实例
data "huaweicloud_workspace_flavors" "test" {
  count = var.desktop_flavor_id == "" ? 1 : 0

  os_type           = var.desktop_flavor_os_type
  vcpus             = var.desktop_flavor_cpu_core_number
  memory            = var.desktop_flavor_memory_size
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行云桌面规格列表查询数据源，仅当 `var.desktop_flavor_id` 为空时创建数据源（即执行云桌面规格列表查询）
- **os_type**：操作系统类型，可选值：Windows、Linux
- **vcpus**：CPU核数，用于筛选规格
- **memory**：内存大小（GB），用于筛选规格
- **availability_zone**：规格所在的可用区，优先使用输入变量中指定的可用区，如未指定则使用数据源查询的第一个可用区

### 4. 通过数据源查询云桌面实例资源创建所需的镜像（data.huaweicloud_images_images）

在TF文件中添加以下脚本以告知Terraform查询符合条件的云桌面镜像：

```hcl
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

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的云桌面镜像信息，用于创建云桌面实例
data "huaweicloud_images_images" "test" {
  count = var.desktop_image_id == "" ? 1 : 0

  name_regex = "WORKSPACE"
  os         = var.desktop_image_os_type
  visibility = var.desktop_image_visibility
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行镜像列表查询数据源，仅当 `var.desktop_image_id` 为空时创建数据源（即执行镜像列表查询）
- **name_regex**：镜像名称的正则表达式，用于筛选云桌面相关镜像
- **os**：操作系统类型，用于筛选镜像
- **visibility**：镜像可见性，market表示云市场镜像

### 5. 通过数据源查询云桌面服务状态（data.huaweicloud_workspace_service）

在TF文件中添加以下脚本以告知Terraform查询当前云桌面服务状态：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下的云桌面服务状态信息，用于判断是否需要创建相关网络资源
data "huaweicloud_workspace_service" "test" {}
```

**参数说明**：
该数据源用于查询当前云桌面服务的状态，如果服务状态为"CLOSED"，则需要创建VPC、子网、安全组等网络资源；如果服务已启用，则可以复用现有的网络资源。

### 6. 创建VPC资源（huaweicloud_vpc）

在TF文件中添加以下脚本以告知Terraform根据云桌面服务状态条件性创建VPC资源：

```hcl
variable "vpc_name" {
  description = "The VPC name"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下根据云桌面服务状态条件性创建VPC资源，用于部署云桌面实例
resource "huaweicloud_vpc" "test" {
  count = data.huaweicloud_workspace_service.test.status == "CLOSED" ? 1 : 0

  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **count**：资源的创建数，仅当云桌面服务状态为"CLOSED"时创建VPC资源
- **name**：VPC名称，通过引用输入变量vpc_name进行赋值
- **cidr**：VPC的CIDR网段，通过引用输入变量vpc_cidr进行赋值

### 7. 创建VPC子网资源（huaweicloud_vpc_subnet）

在TF文件中添加以下脚本以告知Terraform根据云桌面服务状态条件性创建VPC子网资源：

```hcl
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

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下根据云桌面服务状态条件性创建VPC子网资源，用于部署云桌面实例
resource "huaweicloud_vpc_subnet" "test" {
  count = data.huaweicloud_workspace_service.test.status == "CLOSED" ? 1 : 0

  vpc_id     = try(huaweicloud_vpc.test[0].id, null)
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(try(huaweicloud_vpc.test[0].cidr, "192.168.0.0/16"), 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(try(huaweicloud_vpc.test[0].cidr, "192.168.0.0/16"), 8, 0), 1) : var.subnet_gateway_ip
}
```

**参数说明**：
- **count**：资源的创建数，仅当云桌面服务状态为"CLOSED"时创建子网资源
- **vpc_id**：子网所属的VPC的ID，引用前面创建的VPC资源的ID
- **name**：子网的名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网的CIDR网段，如未指定则使用cidrsubnet函数从VPC的CIDR网段中划分一个子网段
- **gateway_ip**：子网的网关IP，如未指定则使用cidrhost函数从子网网段中获取第一个IP地址作为网关IP

### 8. 创建云桌面服务（huaweicloud_workspace_service）

在TF文件中添加以下脚本以告知Terraform根据云桌面服务状态条件性创建云桌面服务资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下根据云桌面服务状态条件性创建云桌面服务资源，用于部署云桌面实例
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
- **count**：资源的创建数，仅当云桌面服务状态为"CLOSED"时创建云桌面服务资源
- **access_mode**：访问模式，使用INTERNET表示通过公网访问
- **vpc_id**：VPC的ID，引用前面创建的VPC资源的ID
- **network_ids**：网络ID列表，引用前面创建的子网资源的ID

### 9. 创建安全组资源（huaweicloud_networking_secgroup）

在TF文件中添加以下脚本以告知Terraform根据云桌面服务状态条件性创建安全组资源：

```hcl
variable "security_group_name" {
  description = "The security group name"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下根据云桌面服务状态条件性创建安全组资源，用于部署云桌面实例
resource "huaweicloud_networking_secgroup" "test" {
  count = data.huaweicloud_workspace_service.test.status == "CLOSED" ? 1 : 0

  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **count**：资源的创建数，仅当云桌面服务状态为"CLOSED"时创建安全组资源
- **name**：安全组的名称，通过引用输入变量security_group_name进行赋值
- **delete_default_rules**：是否删除默认规则，设置为true表示删除默认规则

### 10. 创建安全组规则资源（huaweicloud_networking_secgroup_rule）

在TF文件中添加以下脚本以告知Terraform根据云桌面服务状态条件性创建安全组规则资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下根据云桌面服务状态条件性创建安全组规则资源，用于部署云桌面实例
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
- **count**：资源的创建数，仅当云桌面服务状态为"CLOSED"时创建安全组规则资源
- **security_group_id**：安全组的ID，引用前面创建的安全组资源的ID
- **direction**：规则方向，egress表示出方向流量
- **ethertype**：IP协议版本，IPv4表示IPV4协议
- **remote_ip_prefix**：远端IP地址，0.0.0.0/0表示允许所有IP地址
- **priority**：规则优先级，数值越小优先级越高

### 11. 创建云桌面用户（huaweicloud_workspace_user）

在TF文件中添加以下脚本以告知Terraform创建云桌面用户资源：

```hcl
variable "desktop_user_name" {
  description = "The user name that the cloud desktop used"
  type        = string
}

variable "desktop_user_email" {
  description = "The email address that the user used"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云桌面用户资源，用于部署云桌面实例
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
- **name**：用户名，通过引用输入变量desktop_user_name进行赋值
- **email**：用户邮箱，通过引用输入变量desktop_user_email进行赋值
- **account_expires**：账号过期时间，设置为"0"表示永不过期
- **password_never_expires**：密码是否永不过期，设置为false表示密码有过期时间
- **enable_change_password**：是否允许修改密码，设置为true表示允许修改密码
- **next_login_change_password**：下次登录是否修改密码，设置为true表示下次登录需要修改密码
- **disabled**：是否禁用用户，设置为false表示用户不被禁用

### 12. 创建云桌面实例（huaweicloud_workspace_desktop）

在TF文件中添加以下脚本以告知Terraform创建云桌面实例资源：

```hcl
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
  type = list(object({
    type = string
    size = number
  }))
  default = [
    {
      type = "SSD",
      size = 100,
    },
  ]
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云桌面实例资源
resource "huaweicloud_workspace_desktop" "test" {
  depends_on = [huaweicloud_workspace_user.test]

  flavor_id         = var.desktop_flavor_id == "" ? try([for o in data.huaweicloud_workspace_flavors.test[0].flavors: o.id if !strcontains(lower(o.description), "flexus")][0], null) : var.desktop_flavor_id
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
- **flavor_id**：云桌面规格ID，优先使用输入变量中指定的规格，如未指定则使用数据源查询的第一个非flexus规格
- **image_type**：镜像类型，通过引用输入变量desktop_image_visibility进行赋值
- **image_id**：镜像ID，优先使用输入变量中指定的镜像，如未指定则使用数据源查询的第一个镜像
- **availability_zone**：可用区，优先使用输入变量中指定的可用区，如未指定则使用数据源查询的第一个可用区
- **vpc_id**：VPC的ID，根据云桌面服务状态使用现有VPC或新创建的VPC
- **security_groups**：安全组ID列表，根据云桌面服务状态使用不同的安全组配置
- **nic**：网卡配置块（动态块），根据云桌面服务状态使用不同的网络配置
  - **network_id**：网络的唯一标识符，根据服务状态使用现有网络或新创建的子网
- **name**：云桌面名称，通过引用输入变量cloud_desktop_name进行赋值
- **user_name**：用户名，引用前面创建的云桌面用户资源的名称
- **user_email**：用户邮箱，引用前面创建的云桌面用户资源的邮箱
- **user_group**：用户组名称，通过引用输入变量desktop_user_group_name进行赋值，默认为"users"
- **root_volume**：系统盘配置块
  - **type**：磁盘类型，通过引用输入变量desktop_root_volume_type进行赋值，默认为SSD
  - **size**：磁盘大小，通过引用输入变量desktop_root_volume_size进行赋值，默认为100GB
- **data_volume**：数据盘配置块（动态块）
  - **type**：磁盘类型，通过引用输入变量desktop_data_volumes中的类型值进行赋值
  - **size**：磁盘大小，通过引用输入变量desktop_data_volumes中的大小值进行赋值
- **lifecycle**：生命周期管理，忽略规格、镜像和可用区的变更以避免重建实例

### 13. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，内容如下：

```hcl
vpc_name            = "tf_test_vpc"
vpc_cidr            = "192.168.0.0/16"
subnet_name         = "tf_test_subnet"
security_group_name = "tf_test_security_group"
desktop_user_name   = "tf_test_user"
desktop_user_email  = "test@example.com"
cloud_desktop_name  = "tf-test-desktop"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

> 对于未在`terraform.tfvars`文件中指定的变量，Terraform将使用代码中定义的默认值或在执行时提示用户输入。

### 14. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建云桌面实例
4. 运行 `terraform show` 查看已创建的云桌面实例详情

## 参考信息

- [华为云云桌面产品文档](https://support.huaweicloud.com/workspace/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [云桌面最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/workspace/desktop/basic)

