# 使用Terraform部署按需计费的云桌面

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

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [云桌面服务资源（huaweicloud_workspace_service）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_service)
- [云桌面用户资源（huaweicloud_workspace_user）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_user)
- [云桌面实例资源（huaweicloud_workspace_desktop）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/workspace_desktop)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_workspace_desktop

data.huaweicloud_workspace_flavors
    └── huaweicloud_workspace_desktop

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        ├── huaweicloud_workspace_service
        └── huaweicloud_workspace_desktop

huaweicloud_networking_secgroup
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
  description = "待创建桌面所在可用区的名称，如指定则不执行对应数据源查询"
  type        = string
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
variable "desktop_flavor" {
  description = "云桌面规格，如指定则不执行对应数据源查询"
  type        = string
}

variable "desktop_cpu_core_number" {
  description = "云桌面CPU核数，用于筛选规格"
  type        = number
}

variable "desktop_memory" {
  description = "云桌面内存大小（GB），用于筛选规格"
  type        = number
}

variable "desktop_os_type" {
  description = "云桌面操作系统类型"
  type        = string
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的云桌面规格信息，用于创建云桌面实例
data "huaweicloud_workspace_flavors" "test" {
  count = var.desktop_flavor == "" ? 1 : 0

  vcpus             = var.desktop_cpu_core_number
  memory            = var.desktop_memory
  os_type           = var.desktop_os_type
  availability_zone = var.availability_zone
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行云桌面规格列表查询数据源，仅当 `var.desktop_flavor` 为空时创建数据源（即执行云桌面规格列表查询）
- **vcpus**：CPU核数，用于筛选规格
- **memory**：内存大小（GB），用于筛选规格
- **os_type**：操作系统类型，可选值：windows、linux
- **availability_zone**：规格所在的可用区，用于筛选在售规格

### 4. 创建VPC资源（huaweicloud_vpc）

在TF文件中添加以下脚本以告知Terraform创建VPC资源：

```hcl
variable "vpc_name" {
  description = "VPC名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源，用于部署云桌面实例
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = "192.168.0.0/16"
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量vpc_name进行赋值
- **cidr**：VPC的CIDR网段，本例中使用"192.168.0.0/16"网段

### 5. 创建VPC子网资源（huaweicloud_vpc_subnet）

在TF文件中添加以下脚本以告知Terraform创建VPC子网资源：

```hcl
variable "subnet_name" {
  description = "子网名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源，用于部署云桌面实例
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id            = huaweicloud_vpc.test.id
  name              = var.subnet_name
  cidr              = cidrsubnet(huaweicloud_vpc.test.cidr, 4, 1)
  gateway_ip        = cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 4, 1), 1)
  availability_zone = var.availability_zone
}
```

**参数说明**：
- **vpc_id**：子网所属的VPC的ID，引用前面创建的VPC资源的ID
- **name**：子网的名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网的CIDR网段，使用cidrsubnet函数从VPC的CIDR网段中划分一个子网段
- **gateway_ip**：子网的网关IP，使用cidrhost函数从子网网段中获取第一个IP地址作为网关IP
- **availability_zone**：子网所在的可用区，使用输入变量availability_zone设置

### 6. 创建安全组资源（huaweicloud_networking_secgroup）

在TF文件中添加以下脚本以告知Terraform创建安全组资源：

```hcl
variable "security_group_name" {
  description = "安全组名称"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源，用于部署云桌面实例
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组的名称，通过引用输入变量security_group_name进行赋值
- **delete_default_rules**：是否删除默认规则，设置为true表示删除默认规则

### 7. 创建云桌面服务（huaweicloud_workspace_service）

在TF文件中添加以下脚本以告知Terraform创建云桌面服务资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云桌面服务资源，用于部署云桌面实例
resource "huaweicloud_workspace_service" "test" {
  access_mode = "INTERNET"
  vpc_id      = huaweicloud_vpc.test.id
  network_ids = [
    huaweicloud_vpc_subnet.test.id,
  ]
}
```

**参数说明**：
- **access_mode**：访问模式，使用INTERNET表示通过公网访问
- **vpc_id**：VPC的ID，引用前面创建的VPC资源的ID
- **network_ids**：网络ID列表，引用前面创建的子网资源的ID

### 8. 创建云桌面用户（huaweicloud_workspace_user）

在TF文件中添加以下脚本以告知Terraform创建云桌面用户资源：

```hcl
variable "desktop_user_name" {
  description = "云桌面用户名"
  type        = string
}

variable "desktop_user_email" {
  description = "云桌面用户邮箱"
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

### 9. 创建云桌面实例（huaweicloud_workspace_desktop）

在TF文件中添加以下脚本以告知Terraform创建云桌面实例资源：

```hcl
variable "desktop_image_type" {
  description = "云桌面镜像类型"
  type        = string
}

variable "desktop_image_id" {
  description = "云桌面镜像ID"
  type        = string
}

variable "desktop_user_group_name" {
  description = "云桌面用户组名称"
  type        = string
}

variable "cloud_desktop_name" {
  description = "云桌面实例名称"
  type        = string
}

variable "desktop_root_volume_type" {
  description = "云桌面系统盘类型"
  type        = string
  default     = "SSD"
}

variable "desktop_root_volume_size" {
  description = "云桌面系统盘大小（GB）"
  type        = number
  default     = 100
}

variable "desktop_data_volumes" {
  description = "云桌面数据盘配置列表"
  type = list(object({
    type = string
    size = number
  }))
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云桌面实例资源
resource "huaweicloud_workspace_desktop" "test" {
  depends_on = [huaweicloud_workspace_user.test]

  flavor_id         = var.desktop_flavor != "" ? var.desktop_flavor : try(data.huaweicloud_workspace_flavors.test[0].flavors[0].id, null)
  image_type        = var.desktop_image_type
  image_id          = var.desktop_image_id
  availability_zone = var.availability_zone != "" ? var.availability_zone : try(data.huaweicloud_availability_zones.test[0].names[0], null)
  vpc_id            = huaweicloud_vpc.test.id
  security_groups   = [
    huaweicloud_workspace_service.test.desktop_security_group.0.id,
    huaweicloud_networking_secgroup.test.id,
  ]

  nic {
    network_id = huaweicloud_vpc_subnet.test.id
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
}
```

**参数说明**：
- **flavor_id**：云桌面规格ID，优先使用输入变量中指定的规格，如未指定则使用数据源查询的第一个规格
- **image_type**：镜像类型，通过引用输入变量desktop_image_type进行赋值
- **image_id**：镜像ID，通过引用输入变量desktop_image_id进行赋值
- **availability_zone**：可用区，优先使用输入变量中指定的可用区，如未指定则使用数据源查询的第一个可用区
- **vpc_id**：VPC的ID，引用前面创建的VPC资源的ID
- **security_groups**：安全组ID列表，包含云桌面服务默认安全组和自定义安全组
- **nic**：网卡配置块，指定云桌面实例连接的网络
  - **network_id**：网络的唯一标识符，使用前面创建的子网资源的ID
- **name**：云桌面名称，通过引用输入变量cloud_desktop_name进行赋值
- **user_name**：用户名，引用前面创建的云桌面用户资源的名称
- **user_email**：用户邮箱，引用前面创建的云桌面用户资源的邮箱
- **user_group**：用户组名称，通过引用输入变量desktop_user_group_name进行赋值
- **root_volume**：系统盘配置块
  - **type**：磁盘类型，通过引用输入变量desktop_root_volume_type进行赋值，默认为SSD
  - **size**：磁盘大小，通过引用输入变量desktop_root_volume_size进行赋值，默认为100GB
- **data_volume**：数据盘配置块（动态块）
  - **type**：磁盘类型，通过引用输入变量desktop_data_volumes中的类型值进行赋值
  - **size**：磁盘大小，通过引用输入变量desktop_data_volumes中的大小值进行赋值

### 10. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，内容如下：

```hcl
# 资源命名相关变量
vpc_name            = "workspace-vpc"
subnet_name         = "workspace-subnet"
security_group_name = "workspace-secgroup"
cloud_desktop_name  = "workspace-desktop"

# 云桌面规格和可用区
availability_zone     = "cn-north-4a"
desktop_flavor        = "workspace.x86.medium.win"
desktop_cpu_core_number = 2
desktop_memory          = 4
desktop_os_type         = "windows"

# 云桌面镜像
desktop_image_type     = "market"
desktop_image_id       = "workspace-windows-x64-10-20230628143608"

# 云桌面用户
desktop_user_name      = "workspace-user"
desktop_user_email     = "user@example.com"
desktop_user_group_name = "default-group"

# 云桌面存储
desktop_root_volume_type = "SSD"
desktop_root_volume_size = 100
desktop_data_volumes = [
  {
    type = "SSD"
    size = 100
  }
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="subnet_name=my-subnet"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 11. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建云桌面实例
4. 运行 `terraform show` 查看已创建的云桌面实例详情

## 参考信息

- [华为云云桌面产品文档](https://support.huaweicloud.com/workspace/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [云桌面最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/workspace/desktop/basic)
