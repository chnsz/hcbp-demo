# 使用Terraform部署按需计费的云桌面

## 本最佳实践概述

华为云云桌面（Workspace）是一种基于云计算的桌面虚拟化服务，为企业用户提供安全、便捷的云上办公解决方案。本最佳实践将介绍如何使用Terraform自动化部署按需计费的云桌面实例。

### 应用场景

- 企业需要快速部署和管理云桌面环境
- 需要按需使用和计费的弹性办公解决方案
- 远程办公和移动办公场景
- 临时项目或短期办公需求

### 方案优势

- 自动化部署：使用Terraform实现基础设施即代码
- 按需付费：根据实际使用量计费，降低成本
- 快速交付：快速创建和配置云桌面环境
- 统一管理：集中管理云桌面资源和配置

### 涉及产品

- 云桌面（Workspace）：提供虚拟桌面服务
- 虚拟私有云（VPC）：提供隔离的网络环境
- 统一身份认证服务（IAM）：提供身份认证和权限管理

## 资源/数据源设计

本最佳实践涉及以下主要资源和数据源：

### 数据源

1. **可用区（data.huaweicloud_availability_zones）**
   - 用途：获取可用的可用区信息

2. **云桌面规格（data.huaweicloud_workspace_flavors）**
   - 用途：获取可用的云桌面规格信息

3. **云桌面镜像（data.huaweicloud_workspace_images）**
   - 用途：获取可用的云桌面镜像信息

### 资源

1. **VPC网络（huaweicloud_vpc）**
   - 用途：为云桌面提供网络环境

2. **VPC子网（huaweicloud_vpc_subnet）**
   - 用途：在VPC中划分子网空间

3. **安全组（huaweicloud_networking_secgroup）**
   - 用途：控制云桌面的网络访问

4. **云桌面服务（huaweicloud_workspace_service）**
   - 用途：开通和配置云桌面服务

5. **云桌面用户（huaweicloud_workspace_user）**
   - 用途：创建和管理云桌面用户

6. **云桌面实例（huaweicloud_workspace_desktop）**
   - 用途：提供虚拟桌面环境

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_workspace_desktop

data.huaweicloud_workspace_flavors
    └── huaweicloud_workspace_desktop

data.huaweicloud_workspace_images
    └── huaweicloud_workspace_desktop

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        ├── huaweicloud_workspace_service
        └── huaweicloud_workspace_desktop

huaweicloud_networking_secgroup
    ├── huaweicloud_workspace_desktop
    └── huaweicloud_networking_secgroup_rule

huaweicloud_workspace_user
    └── huaweicloud_workspace_desktop
```

## 详细配置

### 数据源配置

#### 1. 可用区（data.huaweicloud_availability_zones）

获取基于当前provider块中所指定region下的所有可用区信息，用于创建云桌面实例。

```hcl
variable "availability_zone" {
  description = "待创建桌面所在可用区的名称，如指定则不执行对应数据源查询"
  type        = string
}

data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**参数说明**：
- **count**：当 `var.availability_zone` 为空时创建数据源，用于动态获取可用区信息

#### 2. 云桌面规格（data.huaweicloud_workspace_flavors）

获取基于当前provider块中所指定region下所有可用的云桌面规格信息，用于创建云桌面实例。

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

data "huaweicloud_workspace_flavors" "test" {
  count = var.desktop_flavor == "" ? 1 : 0

  vcpus             = var.desktop_cpu_core_number
  memory            = var.desktop_memory
  os_type           = var.desktop_os_type
  availability_zone = var.availability_zone
}
```

**参数说明**：
- **count**：当 `var.desktop_flavor` 为空时创建数据源
- **vcpus**：CPU核数，用于筛选规格
- **memory**：内存大小（GB），用于筛选规格
- **os_type**：操作系统类型，可选值：windows、linux
- **availability_zone**：在售规格的可用区，用于筛选规格

#### 3. 云桌面镜像（data.huaweicloud_workspace_images）

获取可用的云桌面镜像信息，用于创建云桌面实例。

```hcl
variable "desktop_image_type" {
  description = "云桌面镜像类型"
  type        = string
}

variable "desktop_os_type" {
  description = "镜像的操作系统类型"
  type        = string
}

data "huaweicloud_workspace_images" "test" {
  image_type = var.desktop_image_type
  os_type    = var.desktop_os_type
}
```

**参数说明**：
- **image_type**：镜像类型
- **os_type**：操作系统类型

### 资源配置

#### 1. VPC网络（huaweicloud_vpc）

创建VPC网络环境，为云桌面提供网络隔离。

```hcl
variable "vpc_name" {
  description = "VPC名称"
  type        = string
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = "192.168.0.0/16"
}
```

**参数说明**：
- **name**：VPC名称
- **cidr**：VPC网段，格式为CIDR，如192.168.0.0/16

#### 2. VPC子网（huaweicloud_vpc_subnet）

在VPC中创建子网，为云桌面提供网络空间。

```hcl
variable "subnet_name" {
  description = "子网名称"
  type        = string
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id            = huaweicloud_vpc.test.id
  name              = var.subnet_name
  cidr              = cidrsubnet(huaweicloud_vpc.test.cidr, 4, 1)
  gateway_ip        = cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 4, 1), 1)
  availability_zone = var.availability_zone
}
```

**参数说明**：
- **vpc_id**：VPC ID
- **name**：子网名称
- **cidr**：子网网段，使用cidrsubnet函数计算
- **gateway_ip**：网关IP，使用cidrhost函数计算
- **availability_zone**：可用区

#### 3. 安全组（huaweicloud_networking_secgroup）

创建安全组，控制云桌面的网络访问。

```hcl
variable "security_group_name" {
  description = "安全组名称"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组名称
- **delete_default_rules**：是否删除默认规则

#### 4. 云桌面服务（huaweicloud_workspace_service）

开通云桌面服务，这是创建云桌面实例的前置条件。

```hcl
resource "huaweicloud_workspace_service" "test" {
  access_mode = "INTERNET"
  vpc_id      = huaweicloud_vpc.test.id
  network_ids = [
    huaweicloud_vpc_subnet.test.id,
  ]
}
```

**参数说明**：
- **access_mode**：访问模式
- **vpc_id**：VPC ID
- **network_ids**：网络ID列表，用于部署云桌面服务

#### 5. 云桌面用户（huaweicloud_workspace_user）

创建云桌面用户，用于访问云桌面实例。

```hcl
variable "desktop_user_name" {
  description = "云桌面用户名"
  type        = string
}

variable "desktop_user_email" {
  description = "云桌面用户邮箱"
  type        = string
}

resource "huaweicloud_workspace_user" "test" {
  depends_on = [huaweicloud_workspace_service.test]

  name  = var.desktop_user_name
  email = var.desktop_user_email

  account_expires            = "0" # 永不过期
  password_never_expires     = false
  enable_change_password     = true
  next_login_change_password = true
  disabled                   = false
}
```

**参数说明**：
- **name**：用户名
- **email**：用户邮箱
- **account_expires**：账号过期时间
- **password_never_expires**：密码是否永不过期
- **enable_change_password**：是否允许修改密码
- **next_login_change_password**：下次登录是否修改密码
- **disabled**：是否禁用用户

#### 6. 云桌面实例（huaweicloud_workspace_desktop）

创建云桌面实例，提供虚拟桌面环境。

```hcl
variable "desktop_user_group_name" {
  description = "云桌面用户组名称"
  type        = string
}

variable "cloud_desktop_name" {
  description = "云桌面实例名称"
  type        = string
}

variable "desktop_image_id" {
  description = "云桌面镜像ID"
  type        = string
}

variable "desktop_root_volume_type" {
  description = "云桌面系统盘类型"
  type        = string
}

variable "desktop_root_volume_size" {
  description = "云桌面系统盘大小（GB）"
  type        = number
}

variable "desktop_data_volumes" {
  description = "云桌面数据盘配置列表"
  type = list(object({
    type = string
    size = number
  }))
}

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
- **flavor_id**：规格ID，可通过对应数据源（data.huaweicloud_workspace_flavors）获取
- **image_type**：镜像类型
- **image_id**：镜像ID
- **availability_zone**：可用区，可通过对应数据源（data.huaweicloud_availability_zones）获取
- **vpc_id**：VPC ID
- **security_groups**：安全组列表
- **nic**：网卡配置
  * **network_id**：网络ID
- **name**：桌面名称
- **user_name**：用户名
- **user_email**：用户邮箱
- **user_group**：用户组名称
- **root_volume**：系统盘配置
  * **type**：磁盘类型
  * **size**：磁盘大小（GB）
- **data_volume**：数据盘配置（动态块）
  * **type**：磁盘类型
  * **size**：磁盘大小（GB）

### 可扩展配置

#### 1. 安全组规则（huaweicloud_networking_secgroup_rule）

创建用于访问云桌面的入方向规则。

```hcl
resource "huaweicloud_networking_secgroup_rule" "login_and_web" {
  security_group_id = huaweicloud_networking_secgroup.workspace.id
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  ports             = "22,80,443,3389"
  remote_ip_prefix  = "192.168.1.0/24"  # 替换为您的办公网络IP段
  description       = "Allow access from office network"
}
```

**参数说明**：
- **security_group_id**：安全组ID
- **direction**：规则方向
- **ethertype**：网络协议版本
- **protocol**：协议类型
- **ports**：端口范围，支持配置多个端口，如"22,80,443,3389"
- **remote_ip_prefix**：允许访问的IP范围，CIDR格式
- **description**：规则描述

> 注意：该规则适用于需要远程登录（22/3389）、公网ping（ICMP需单独配置）、以及网站服务（80/443）的云服务器场景。
  该规则示例中使用了示例IP段（192.168.1.0/24），实际部署时请替换为您企业办公网络的具体IP段。
  建议遵循最小权限原则，只开放必要的端口和IP范围，以提高安全性。

## 部署流程

1. 创建VPC和子网
2. 配置安全组规则
3. 开通云桌面服务
4. 创建云桌面实例

## 操作步骤

1. **准备工作**
   - 安装Terraform
   - 配置华为云认证信息
   - 创建工作目录

2. **创建Terraform配置文件**
   ```bash
   touch main.tf
   touch variables.tf
   ```

3. **初始化和部署**
   ```bash
   terraform init
   terraform plan
   terraform apply
   ```

4. **验证部署**
   - 登录华为云控制台
   - 检查云桌面实例状态
   - 测试远程连接

## 注意事项

1. **成本控制**：
   - 选择合适的实例规格
   - 及时关闭不使用的实例
   - 合理规划资源使用

2. **网络规划**：
   - 合理规划IP地址段
   - 配置必要的安全组规则
   - 确保网络连通性

3. **安全配置**：
   - 启用安全组防护
   - 配置访问控制策略
   - 定期更新系统补丁

## 最佳实践效果

通过本最佳实践的实施，您将获得：

1. 自动化部署的云桌面环境
2. 按需计费的资源使用模式
3. 安全可控的网络访问策略
4. 可重复使用的Terraform配置

## 参考信息

- [华为云云桌面产品文档](https://support.huaweicloud.com/workspace/index.html)
- [Terraform华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [云桌面最佳实践](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/workspace/desktop/basic)
