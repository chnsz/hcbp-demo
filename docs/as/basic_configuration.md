# 使用Terraform配置弹性伸缩服务

## 概述

华为云弹性伸缩服务（Auto Scaling）是一种自动调整计算资源的服务，能够根据业务需求和策略，自动调整弹性计算实例的数量。本最佳实践将介绍如何使用Terraform配置基础的弹性伸缩服务。

### 应用场景

- 根据业务负载自动调整资源数量
- 降低手动管理服务器的运维成本
- 提高应用的高可用性
- 优化资源使用成本

### 方案优势

- 自动化：根据策略自动调整资源数量
- 经济性：按需使用资源，降低成本
- 可靠性：自动维护资源池的健康状态
- 灵活性：支持多种伸缩策略和触发条件

### 涉及服务

- 弹性伸缩服务（AS）：提供资源自动伸缩能力
- 虚拟私有云（VPC）：提供网络环境
- 统一身份认证服务（IAM）：提供身份认证和权限管理
- 密钥管理服务（KPS）：提供密钥对管理能力

## 资源/数据源设计

本最佳实践涉及以下主要资源和数据源：

### 数据源

1. **可用区（data.huaweicloud_availability_zones）**
   - 用途：获取可用的可用区信息

2. **镜像（data.huaweicloud_images_image）**
   - 用途：获取弹性伸缩实例使用的操作系统镜像

3. **实例规格（data.huaweicloud_compute_flavors）**
   - 用途：获取指定可用区下的实例规格信息

### 资源

1. **安全组（huaweicloud_networking_secgroup）**
   - 用途：控制实例的网络访问

2. **密钥对（huaweicloud_kps_keypair）**
   - 用途：为弹性伸缩实例提供SSH密钥认证

3. **AS配置（huaweicloud_as_configuration）**
   - 用途：定义弹性伸缩实例的模板

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones.test
    └── data.huaweicloud_compute_flavors.test
        └── huaweicloud_as_configuration.acc_as_config

data.huaweicloud_images_image.test
    └── huaweicloud_as_configuration.acc_as_config

huaweicloud_networking_secgroup.test
    └── huaweicloud_as_configuration.acc_as_config

huaweicloud_kps_keypair.acc_key
    └── huaweicloud_as_configuration.acc_as_config
```

## 详细配置

### 数据源配置

#### 1. 可用区（data.huaweicloud_availability_zones）

获取默认region（默认继承当前provider块中所指定的region）下所有的可用区信息，用于获取实例规格。

```hcl
data "huaweicloud_availability_zones" "test" {}
```

**参数说明**：
无需参数配置，默认获取当前区域下所有可用区信息。

#### 2. 镜像（data.huaweicloud_images_image）

获取默认region（默认继承当前provider块中所指定的region）下的操作系统镜像信息，用于创建弹性伸缩实例。

```hcl
data "huaweicloud_images_image" "test" {
  name        = "Ubuntu 18.04 server 64bit"
  visibility  = "public"
  most_recent = true
}
```

**参数说明**：
- **name**：镜像名称
- **visibility**：镜像可见性
- **most_recent**：是否使用最新版本的镜像

#### 3. 实例规格（data.huaweicloud_compute_flavors）

获取默认region（默认继承当前provider块中所指定的region）下指定可用区的实例规格信息，用于创建弹性伸缩实例。

```hcl
data "huaweicloud_compute_flavors" "test" {
  availability_zone = data.huaweicloud_availability_zones.test.names[0]
  performance_type  = "normal"
  cpu_core_count    = 2
  memory_size       = 4
}
```

**参数说明**：
- **availability_zone**：可用区名称
- **performance_type**：性能类型
- **cpu_core_count**：CPU核心数
- **memory_size**：内存大小（GB）

### 资源配置

#### 1. 安全组（huaweicloud_networking_secgroup）

在默认region（默认继承当前provider块中所指定的region）下创建安全组，控制实例的网络访问。

```hcl
resource "huaweicloud_networking_secgroup" "test" {
  name                 = "test-secgroup-demo"
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组名称
- **delete_default_rules**：是否删除默认规则

#### 2. 密钥对（huaweicloud_kps_keypair）

在默认region（默认继承当前provider块中所指定的region）下创建密钥对，为弹性伸缩实例提供SSH密钥认证。

```hcl
variable "public_key" {
  description = "The public key for the keypair"
  type        = string
}

resource "huaweicloud_kps_keypair" "acc_key" {
  name       = "test-keypair-demo"
  public_key = var.public_key
}
```

**参数说明**：
- **name**：密钥对名称
- **public_key**：公钥内容

#### 3. AS配置（huaweicloud_as_configuration）

在默认region（默认继承当前provider块中所指定的region）下创建AS配置，定义弹性伸缩实例的模板。

```hcl
resource "huaweicloud_as_configuration" "acc_as_config" {
  scaling_configuration_name = "test-as-configuration-demo"
  instance_config {
    image              = data.huaweicloud_images_image.test.id
    flavor             = data.huaweicloud_compute_flavors.test.ids[0]
    key_name           = huaweicloud_kps_keypair.acc_key.id
    security_group_ids = [huaweicloud_networking_secgroup.test.id]

    metadata = {
      some_key = "some_value"
    }
    user_data = <<EOT
#!/bin/sh
echo "Hello World! The time is now $(date -R)!" | tee /root/output.txt
EOT

    disk {
      size        = 40
      volume_type = "SSD"
      disk_type   = "SYS"
    }

    public_ip {
      eip {
        ip_type = "5_bgp"
        bandwidth {
          size          = 10
          share_type    = "PER"
          charging_mode = "traffic"
        }
      }
    }
  }
}
```

**参数说明**：
- **scaling_configuration_name**：配置名称
- **instance_config**：实例配置
  * **image**：镜像ID
  * **flavor**：实例规格
  * **key_name**：密钥对ID
  * **security_group_ids**：安全组ID列表
  * **metadata**：元数据配置
  * **user_data**：实例启动脚本
  * **disk**：磁盘配置
    - **size**：磁盘大小（GB）
    - **volume_type**：磁盘类型
    - **disk_type**：磁盘用途
  * **public_ip**：公网IP配置
    - **eip**：弹性IP配置
      + **ip_type**：公网IP类型，如5_bgp表示动态BGP
      + **bandwidth**：带宽配置
        - **size**：带宽大小，单位为Mbps
        - **share_type**：带宽类型，PER表示独享带宽
        - **charging_mode**：计费模式，traffic表示按流量计费

## 部署流程

1. 配置安全组
2. 创建密钥对
3. 创建AS配置
4. 验证配置

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
   - 检查AS配置状态
   - 验证配置参数

## 注意事项

1. **实例配置**：
   - 选择合适的实例规格
   - 确保镜像可用性
   - 正确配置密钥对

2. **安全配置**：
   - 配置正确的安全组规则
   - 确保密钥对安全保存

3. **成本控制**：
   - 合理配置实例规格
   - 及时清理不需要的资源
   - 选择合适的计费方式

## 最佳实践效果

通过本最佳实践的实施，您将获得：

1. 自动化部署的AS配置环境
2. 标准化的实例配置模板
3. 可重复使用的Terraform配置
4. 标准化的运维管理流程

## 参考信息

- [华为云弹性伸缩服务产品文档](https://support.huaweicloud.com/intl/zh-cn/as/index.html)
- [Terraform华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [AS最佳实践](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/as/configuration-basic)
