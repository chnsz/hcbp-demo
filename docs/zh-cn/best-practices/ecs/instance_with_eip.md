# 部署绑定EIP的实例

## 应用场景

弹性云服务器（Elastic Cloud Server, ECS）是由CPU、内存、操作系统、云硬盘组成的基础的计算组件，为您的应用提供可靠、安全、灵活、高效的计算环境。ECS服务支持多种实例规格和操作系统，满足不同规模和场景的计算需求。

绑定EIP的ECS实例是ECS服务中的重要功能，通过为ECS实例绑定弹性公网IP（EIP），使实例能够直接访问互联网，同时也能被互联网用户访问。这种配置适用于需要公网访问的Web服务器、API服务器、数据库服务器等场景。本最佳实践将介绍如何使用Terraform自动化部署绑定EIP的ECS实例，包括网络环境创建、安全组配置、实例创建和EIP绑定。

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
- [安全组规则资源（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [ECS实例资源（huaweicloud_compute_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [EIP资源（huaweicloud_vpc_eip）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [EIP绑定资源（huaweicloud_compute_eip_associate）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_eip_associate)

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
    ├── huaweicloud_networking_secgroup_rule.test
    └── huaweicloud_compute_instance.test

huaweicloud_vpc_eip.test
    └── huaweicloud_compute_eip_associate.test

huaweicloud_compute_instance.test
    └── huaweicloud_compute_eip_associate.test
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
- **步骤8**：创建ECS实例资源

完成上述步骤后，继续执行本最佳实践的后续步骤。

### 3. 创建安全组规则

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建安全组规则资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组规则资源
resource "huaweicloud_networking_secgroup_rule" "test" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "egress"
  ethertype         = "IPv4"
  remote_ip_prefix  = "0.0.0.0/0"
  priority          = 1
}
```

**参数说明**：
- **security_group_id**：安全组ID，通过引用安全组资源（huaweicloud_networking_secgroup.test）的ID进行赋值
- **direction**：方向，设置为"egress"表示出站规则
- **ethertype**：以太网类型，设置为"IPv4"表示IPv4协议
- **remote_ip_prefix**：远程IP前缀，设置为"0.0.0.0/0"表示允许访问所有IP
- **priority**：优先级，设置为1表示高优先级

### 4. 创建EIP

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建EIP资源：

```hcl
variable "associate_eip_address" {
  description = "要绑定到ECS实例的EIP地址"
  type        = string
  default     = ""
}

variable "eip_type" {
  description = "EIP类型"
  type        = string
  default     = "5_bgp"
}

variable "bandwidth_name" {
  description = "EIP带宽名称"
  type        = string
  default     = ""
}

variable "bandwidth_size" {
  description = "带宽大小"
  type        = number
  default     = 5
}

variable "bandwidth_share_type" {
  description = "带宽共享类型"
  type        = string
  default     = "PER"
}

variable "bandwidth_charge_mode" {
  description = "带宽计费模式"
  type        = string
  default     = "traffic"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建EIP资源
resource "huaweicloud_vpc_eip" "test" {
  count = var.associate_eip_address == "" ? 1 : 0

  publicip {
    type = var.eip_type
  }

  bandwidth {
    name        = var.bandwidth_name
    size        = var.bandwidth_size
    share_type  = var.bandwidth_share_type
    charge_mode = var.bandwidth_charge_mode
  }

  lifecycle {
    precondition {
      condition     = var.associate_eip_address != "" || var.bandwidth_name != ""
      error_message = "The bandwidth name must be a non-empty string if the EIP address is not provided."
    }
  }
}
```

**参数说明**：
- **count**：资源的创建数，用于控制是否创建EIP资源，仅当 `var.associate_eip_address` 为空时创建资源
- **publicip.type**：公网IP类型，通过引用输入变量eip_type进行赋值
- **bandwidth.name**：带宽名称，通过引用输入变量bandwidth_name进行赋值
- **bandwidth.size**：带宽大小，通过引用输入变量bandwidth_size进行赋值
- **bandwidth.share_type**：带宽共享类型，通过引用输入变量bandwidth_share_type进行赋值
- **bandwidth.charge_mode**：带宽计费模式，通过引用输入变量bandwidth_charge_mode进行赋值
- **lifecycle.precondition**：生命周期前置条件，确保带宽名称不为空

### 5. 绑定EIP

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建EIP绑定资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建EIP绑定资源
resource "huaweicloud_compute_eip_associate" "test" {
  instance_id = huaweicloud_compute_instance.test.id
  public_ip   = var.associate_eip_address == "" ? try(huaweicloud_vpc_eip.test[0].address, null) : var.associate_eip_address
}
```

**参数说明**：
- **public_ip**：公网IP地址，优先使用输入变量，如果为空则使用EIP资源的地址
- **instance_id**：实例ID，通过引用ECS实例资源（huaweicloud_compute_instance.test）的ID进行赋值

### 6. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 网络配置
vpc_name    = "tf_test_vpc"
vpc_cidr    = "192.168.0.0/16"
subnet_name = "tf_test_subnet"

# 安全组配置
security_group_name = "tf_test_security_group"

# ECS实例配置
instance_name          = "tf_test_instance"
administrator_password = "YourPasswordInput!"

# EIP配置
eip_type               = "5_bgp"
bandwidth_name         = "tf_test_bandwidth"
bandwidth_size         = 5
bandwidth_share_type   = "PER"
bandwidth_charge_mode  = "traffic"
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

### 7. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建绑定EIP的ECS实例
4. 运行 `terraform show` 查看已创建的绑定EIP的ECS实例

## 参考信息

- [华为云ECS产品文档](https://support.huaweicloud.com/ecs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [ECS绑定EIP的实例最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ecs/instance-associate-eip)
