# 部署基础弹性云服务器

## 应用场景

弹性云服务器（Elastic Cloud Server，ECS）是由CPU、内存、操作系统、云硬盘组成的基础的计算组件。弹性云服务器创建成功后，您就可以像使用自己的本地PC或物理服务器一样，在云上使用弹性云服务器。华为云提供了多种类型的弹性云服务器，可满足不同的使用场景。在创建之前，您需要根据实际的应用场景确认弹性云服务器的规格类型，镜像类型，磁盘种类等参数，并选择合适的网络参数和安全组规则。本最佳实践将介绍如何使用Terraform自动化部署一个基础的ECS实例，包括VPC、子网和安全组的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ECS规格列表查询数据源（data.huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [IMS镜像列表查询数据源（data.huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [ECS实例资源（huaweicloud_compute_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── data.huaweicloud_compute_flavors
        └── data.huaweicloud_images_images
            └── huaweicloud_compute_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_compute_instance

huaweicloud_networking_secgroup
    └── huaweicloud_compute_instance
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询ECS实例资源创建所需的可用区

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建ECS实例：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建ECS实例
data "huaweicloud_availability_zones" "test" {}
```

**参数说明**：
此数据源无需额外参数，默认查询当前区域下所有可用的可用区信息。

### 3. 通过数据源查询ECS实例资源创建所需的规格

在TF文件中添加以下脚本以告知Terraform查询符合条件的ECS规格：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的ECS规格信息，用于创建ECS实例
data "huaweicloud_compute_flavors" "test" {
  availability_zone = data.huaweicloud_availability_zones.test.names[0]
  performance_type  = "normal"
  cpu_core_count    = 2
  memory_size       = 4
}
```

**参数说明**：
- **availability_zone**：规格所在的可用区，使用前面查询的可用区数据源的第一个可用区
- **performance_type**：规格的类型，设置为"normal"表示标准类型
- **cpu_core_count**：CPU核心数，设置为2核
- **memory_size**：内存大小（GB），设置为4GB

### 4. 通过数据源查询ECS实例资源创建所需的镜像

在TF文件中添加以下脚本以告知Terraform查询符合条件的镜像：

```hcl
# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有符合特定条件的IMS镜像信息，用于创建ECS实例
data "huaweicloud_images_images" "test" {
  flavor_id  = try(data.huaweicloud_compute_flavors.test.flavors[0].id, "")
  visibility = "public"
  os         = "Ubuntu"
}
```

**参数说明**：
- **flavor_id**：镜像支持的规格ID，使用前面查询的规格数据源的第一个规格ID，如果规格数据源查询失败则使用空字符串
- **visibility**：镜像的可见性，设置为"public"表示公共镜像
- **os**：镜像的操作系统类型，设置为"Ubuntu"操作系统

### 5. 创建VPC资源

在TF文件中添加以下脚本以告知Terraform创建VPC资源：

```hcl
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源，用于部署ECS实例
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = "192.168.0.0/16"
}
```

**参数说明**：
- **name**：VPC的名称，通过引用输入变量vpc_name进行赋值
- **cidr**：VPC的CIDR网段，设置为"192.168.0.0/16"网段

### 6. 创建VPC子网资源

在TF文件中添加以下脚本以告知Terraform创建VPC子网资源：

```hcl
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源，用于部署ECS实例
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**参数说明**：
- **vpc_id**：子网所属的VPC的ID，引用前面创建的VPC资源的ID
- **name**：子网的名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网的CIDR网段，使用cidrsubnet函数从VPC的CIDR网段中划分一个子网段
- **gateway_ip**：子网的网关IP，使用cidrhost函数从子网网段中获取第一个IP地址作为网关IP

### 7. 创建安全组资源

在TF文件中添加以下脚本以告知Terraform创建安全组资源：

```hcl
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源，用于部署ECS实例
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组的名称，通过引用输入变量security_group_name进行赋值
- **delete_default_rules**：是否删除默认规则，设置为true表示删除默认规则

### 8. 创建ECS实例资源

在TF文件中添加以下脚本以告知Terraform创建ECS实例资源：

```hcl
variable "instance_name" {
  description = "The name of the ECS instance"
  type        = string
}

variable "administrator_password" {
  description = "The password of the administrator"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ECS实例资源
resource "huaweicloud_compute_instance" "basic" {
  name               = var.instance_name
  availability_zone  = data.huaweicloud_availability_zones.test.names[0]
  flavor_id          = try(data.huaweicloud_compute_flavors.test.flavors[0].id, "")
  image_id           = try(data.huaweicloud_images_images.test.images[0].id, "")
  security_group_ids = [
    huaweicloud_networking_secgroup.test.id
  ]

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }
  
  admin_pass = var.administrator_password

  lifecycle {
    ignore_changes = [
      admin_pass,
    ]
  }
}
```

**参数说明**：
- **name**：ECS实例的名称，通过引用输入变量instance_name进行赋值
- **availability_zone**：ECS实例所在的可用区，使用前面查询的可用区数据源的第一个可用区
- **flavor_id**：ECS实例所使用规格的ID，使用前面查询的规格数据源的第一个规格ID，如果规格数据源查询失败则使用空字符串
- **image_id**：ECS实例所使用镜像的ID，使用前面查询的镜像数据源的第一个镜像ID，如果镜像数据源查询失败则使用空字符串
- **security_group_ids**：与ECS实例关联的安全组ID列表，使用前面创建的安全组资源的ID
- **network**：网络配置块，指定ECS实例连接的网络
  - **uuid**：网络的唯一标识符，使用前面创建的子网资源的ID
- **admin_pass**：ECS实例的管理员密码，通过引用输入变量administrator_password进行赋值
- **lifecycle**：生命周期配置块，用于控制资源的生命周期行为
  - **ignore_changes**：指定在后续应用中要忽略的属性变更，设置为忽略admin_pass的变更

### 9. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 华为云认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# VPC配置
vpc_name = "tf_test_vpc"

# 子网配置
subnet_name = "tf_test_subnet"

# 安全组配置
security_group_name = "tf_test_secgroup"

# ECS实例配置
instance_name = "tf_test_instance"
administrator_password = "YourPassword123!"
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

### 10. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建ECS实例
4. 运行 `terraform show` 查看已创建的ECS实例详情

## 参考信息

- [华为云ECS产品文档](https://support.huaweicloud.com/ecs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [ECS最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ecs)
