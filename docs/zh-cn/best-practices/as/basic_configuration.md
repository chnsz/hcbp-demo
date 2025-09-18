# 部署基础弹性伸缩配置

## 应用场景

华为云弹性伸缩服务（Auto Scaling）是一种自动调整计算资源的服务，能够根据业务需求和策略，自动调整弹性计算实例的数量。通过配置AS配置，可以定义弹性伸缩实例的模板，包括镜像、规格、安全组等配置，为后续创建伸缩组提供基础。本最佳实践将介绍如何使用Terraform自动化部署基础AS配置，包括安全组、密钥对和AS配置的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [镜像查询数据源（data.huaweicloud_images_image）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_image)
- [计算规格列表查询数据源（data.huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)

### 资源

- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [密钥对资源（huaweicloud_kps_keypair）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/kps_keypair)
- [AS配置资源（huaweicloud_as_configuration）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/as_configuration)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── data.huaweicloud_compute_flavors
        └── huaweicloud_as_configuration

data.huaweicloud_images_image
    └── huaweicloud_as_configuration

huaweicloud_networking_secgroup
    └── huaweicloud_as_configuration

huaweicloud_kps_keypair
    └── huaweicloud_as_configuration
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询AS配置资源创建所需的可用区

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建AS配置：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建AS配置
data "huaweicloud_availability_zones" "test" {}
```

**参数说明**：
此数据源无需额外参数，默认查询当前区域下所有可用的可用区信息。

### 3. 通过数据源查询AS配置资源创建所需的镜像

在TF文件中添加以下脚本以告知Terraform查询符合条件的镜像：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的镜像信息，用于创建AS配置
data "huaweicloud_images_image" "test" {
  name        = "Ubuntu 18.04 server 64bit"
  visibility  = "public"
  most_recent = true
}
```

**参数说明**：
- **name**：镜像名称，设置为"Ubuntu 18.04 server 64bit"
- **visibility**：镜像可见性，设置为"public"表示公共镜像
- **most_recent**：是否使用最新版本的镜像，设置为true表示使用最新版本

### 4. 通过数据源查询AS配置资源创建所需的实例规格

在TF文件中添加以下脚本以告知Terraform查询符合条件的实例规格：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的实例规格信息，用于创建AS配置
data "huaweicloud_compute_flavors" "test" {
  availability_zone = data.huaweicloud_availability_zones.test.names[0]
  performance_type  = "normal"
  cpu_core_count    = 2
  memory_size       = 4
}
```

**参数说明**：
- **availability_zone**：实例规格所在的可用区，使用可用区列表查询数据源的第一个可用区
- **performance_type**：性能类型，设置为"normal"表示标准型
- **cpu_core_count**：CPU核心数，设置为2核
- **memory_size**：内存大小（GB），设置为4GB

### 5. 创建安全组资源

在TF文件中添加以下脚本以告知Terraform创建安全组资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源，用于部署AS配置
resource "huaweicloud_networking_secgroup" "test" {
  name                 = "test-secgroup-demo"
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组名称，设置为"test-secgroup-demo"
- **delete_default_rules**：是否删除默认规则，设置为true表示删除默认规则

### 6. 创建密钥对资源

在TF文件中添加以下脚本以告知Terraform创建密钥对资源：

```hcl
variable "public_key" {
  description = "The public key for the keypair"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建密钥对资源，用于部署AS配置
resource "huaweicloud_kps_keypair" "acc_key" {
  name       = "test-keypair-demo"
  public_key = var.public_key
}
```

**参数说明**：
- **name**：密钥对名称，设置为"test-keypair-demo"
- **public_key**：公钥内容，通过引用输入变量public_key进行赋值

### 7. 创建AS配置资源

在TF文件中添加以下脚本以告知Terraform创建AS配置资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建AS配置资源
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
- **scaling_configuration_name**：AS配置名称，设置为"test-as-configuration-demo"
- **instance_config**：实例配置块
  - **image**：镜像ID，使用镜像查询数据源的ID
  - **flavor**：实例规格，使用实例规格查询数据源的第一个规格ID
  - **key_name**：密钥对ID，引用前面创建的密钥对资源的ID
  - **security_group_ids**：安全组ID列表，引用前面创建的安全组资源的ID
  - **metadata**：元数据配置，设置键值对
  - **user_data**：实例启动脚本，用于初始化实例
  - **disk**：磁盘配置块
    - **size**：磁盘大小（GB），设置为40GB
    - **volume_type**：磁盘类型，设置为"SSD"
    - **disk_type**：磁盘用途，设置为"SYS"表示系统盘
  - **public_ip**：公网IP配置块
    - **eip**：弹性公网IP配置
      - **ip_type**：公网IP类型，设置为"5_bgp"
      - **bandwidth**：带宽配置块
        - **size**：带宽大小，设置为10Mbps
        - **share_type**：带宽类型，设置为"PER"表示独享
        - **charging_mode**：计费模式，设置为"traffic"表示按流量计费

### 8. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 华为云认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 密钥对配置
public_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC..."
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="public_key=your-public-key"`
2. 环境变量：`export TF_VAR_public_key=your-public-key`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 9. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建AS配置
4. 运行 `terraform show` 查看已创建的AS配置详情

## 参考信息

- [华为云弹性伸缩产品文档](https://support.huaweicloud.com/as/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [AS基础配置最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/as)
