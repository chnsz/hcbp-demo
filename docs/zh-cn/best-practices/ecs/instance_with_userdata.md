# 部署实例通过UserData执行脚本

## 应用场景

弹性云服务器（Elastic Cloud Server, ECS）是由CPU、内存、操作系统、云硬盘组成的基础的计算组件，为您的应用提供可靠、安全、灵活、高效的计算环境。ECS服务支持多种实例规格和操作系统，满足不同规模和场景的计算需求。

通过UserData执行脚本是ECS实例的重要功能，允许在实例启动时自动执行自定义脚本，实现自动化配置、软件安装、服务启动等操作。这种配置适用于需要自动化部署、环境初始化、应用配置等场景。本最佳实践将介绍如何使用Terraform自动化部署通过UserData执行脚本的ECS实例，包括网络环境创建、安全组配置、密钥对创建、实例创建和脚本执行。

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
- [密钥对资源（huaweicloud_kps_keypair）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/kps_keypair)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones.test
    └── huaweicloud_compute_instance.test

data.huaweicloud_compute_flavors.test
    └── huaweicloud_compute_instance.test

data.huaweicloud_images_images.test
    └── huaweicloud_compute_instance.test

huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test
        └── huaweicloud_compute_instance.test

huaweicloud_networking_secgroup.test
    └── huaweicloud_compute_instance.test

huaweicloud_kps_keypair.test
    └── huaweicloud_compute_instance.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 前置资源准备

本最佳实践需要先创建VPC、子网、安全组和ECS实例等前置资源。请按照[部署基础弹性云服务器](simple_instance.md)最佳实践中的以下步骤进行准备：

- **步骤2**：通过数据源查询ECS实例资源创建所需的可用区
- **步骤3**：通过数据源查询ECS实例资源创建所需的规格
- **步骤4**：通过数据源查询ECS实例资源创建所需的镜像
- **步骤5**：创建VPC资源
- **步骤6**：创建VPC子网资源
- **步骤7**：创建安全组资源

完成上述步骤后，继续执行本最佳实践的后续步骤。

### 3. 创建密钥对

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建密钥对资源：

```hcl
variable "keypair_name" {
  description = "密钥对名称"
  type        = string
}

variable "keypair_public_key" {
  description = "密钥对的公钥"
  type        = string
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建密钥对资源
resource "huaweicloud_kps_keypair" "test" {
  name       = var.keypair_name
  public_key = var.keypair_public_key
}
```

**参数说明**：
- **name**：密钥对名称，通过引用输入变量keypair_name进行赋值
- **public_key**：公钥内容，通过引用输入变量keypair_public_key进行赋值，如果为null则系统自动生成

### 4. 创建ECS实例并配置UserData

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建ECS实例资源并配置UserData：

```hcl
variable "instance_name" {
  description = "ECS实例名称"
  type        = string
}

variable "instance_user_data" {
  description = "ECS实例的UserData脚本"
  type        = string
}

variable "security_group_ids" {
  description = "ECS实例的安全组ID列表"
  type        = list(string)
  default     = []
  nullable    = false
}

variable "security_group_names" {
  description = "ECS实例的安全组名称列表"
  type        = list(string)
  default     = []
  nullable    = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ECS实例资源
resource "huaweicloud_compute_instance" "test" {
  name               = var.instance_name
  image_id           = var.instance_image_id == "" ? try(data.huaweicloud_images_images.test[0].images[0].id, null) : var.instance_image_id
  flavor_id          = var.instance_flavor_id == "" ? try(data.huaweicloud_compute_flavors.test[0].ids[0], null) : var.instance_flavor_id
  security_group_ids = length(var.security_group_ids) == 0 ? huaweicloud_networking_secgroup.test[*].id : var.security_group_ids
  key_pair           = huaweicloud_kps_keypair.test.name
  availability_zone  = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
  user_data          = var.instance_user_data

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }

  # 如果修改以下字段，请从lifecycle块中删除相应字段
  lifecycle {
    ignore_changes = [
      image_id,
      flavor_id,
      availability_zone
    ]
  }
}
```

**参数说明**：
- **name**：实例名称，通过引用输入变量instance_name进行赋值
- **image_id**：镜像ID，优先使用输入变量，如果为空则使用镜像列表查询数据源的第一个结果
- **flavor_id**：规格ID，优先使用输入变量，如果为空则使用ECS规格列表查询数据源的第一个结果
- **security_group_ids**：安全组ID列表，优先使用输入变量，如果为空则使用创建的安全组资源ID列表
- **key_pair**：密钥对名称，通过引用密钥对资源（huaweicloud_kps_keypair.test）的名称进行赋值
- **availability_zone**：可用区，优先使用输入变量，如果为空则使用可用区列表查询数据源的第一个结果
- **user_data**：UserData脚本，通过引用输入变量instance_user_data进行赋值
- **network.uuid**：网络UUID，通过引用VPC子网资源（huaweicloud_vpc_subnet.test）的ID进行赋值
- **lifecycle.ignore_changes**：生命周期忽略变更，忽略image_id、flavor_id和availability_zone的变更

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 网络配置
vpc_name    = "tf_test_vpc_with_ecs_instance"
vpc_cidr    = "192.168.0.0/16"
subnet_name = "tf_test_subnet_with_ecs_instance"

# 安全组配置
security_group_names = ["tf_test_seg1_with_ecs_instance", "tf_test_seg2_with_ecs_instance"]

# ECS实例配置
instance_name        = "tf_test_with_userdata"
keypair_name         = "tf_test_keypair_with_ecs_instance"
instance_user_data   = <<EOF
#!/bin/bash
echo "Hello, World!" > /home/terraform.txt
EOF
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="instance_name=my-instance"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建通过UserData执行脚本的ECS实例
4. 运行 `terraform show` 查看已创建的ECS实例

## 参考信息

- [华为云ECS产品文档](https://support.huaweicloud.com/ecs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [ECS最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ecs)
