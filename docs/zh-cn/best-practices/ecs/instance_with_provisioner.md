# 部署实例通过provisioner远程登录

## 应用场景

弹性云服务器（Elastic Cloud Server, ECS）是由CPU、内存、操作系统、云硬盘组成的基础的计算组件，为您的应用提供可靠、安全、灵活、高效的计算环境。ECS服务支持多种实例规格和操作系统，满足不同规模和场景的计算需求。

通过provisioner远程登录的ECS实例是ECS服务中的高级功能，通过Terraform的provisioner机制，可以在ECS实例创建完成后自动执行远程命令，实现实例的自动化配置和初始化。这种配置适用于需要自动化部署、配置管理、应用安装等场景。本最佳实践将介绍如何使用Terraform自动化部署通过provisioner远程登录的ECS实例，包括密钥对创建、网络环境配置、EIP绑定和远程命令执行。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ECS规格列表查询数据源（data.huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [镜像列表查询数据源（data.huaweicloud_images_images）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_images)

### 资源

- [密钥对资源（huaweicloud_kps_keypair）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/kps_keypair)
- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则资源（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [EIP资源（huaweicloud_vpc_eip）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [ECS实例资源（huaweicloud_compute_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [EIP绑定资源（huaweicloud_compute_eip_associate）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_eip_associate)
- [空资源（null_resource）](https://registry.terraform.io/providers/hashicorp/null/latest/docs/resources/resource)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones.test
    └── huaweicloud_compute_instance.test

data.huaweicloud_compute_flavors.test
    └── huaweicloud_compute_instance.test

data.huaweicloud_images_images.test
    └── huaweicloud_compute_instance.test

huaweicloud_kps_keypair.test
    └── huaweicloud_compute_instance.test

huaweicloud_vpc.test
    └── huaweicloud_vpc_subnet.test
        └── huaweicloud_compute_instance.test

huaweicloud_networking_secgroup.test
    ├── huaweicloud_networking_secgroup_rule.test
    └── huaweicloud_compute_instance.test

huaweicloud_vpc_eip.test
    └── huaweicloud_compute_eip_associate.test
        └── null_resource.test

huaweicloud_compute_instance.test
    └── huaweicloud_compute_eip_associate.test
        └── null_resource.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 前置资源准备

本最佳实践需要先创建VPC、子网、安全组等前置资源。请按照[部署基础弹性云服务器](simple_instance.md)最佳实践中的以下步骤进行准备：

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
  description = "SSH密钥对名称"
  type        = string
}

variable "private_key_path" {
  description = "私钥文件路径"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建密钥对资源
resource "huaweicloud_kps_keypair" "test" {
  name     = var.keypair_name
  key_file = var.private_key_path
}
```

**参数说明**：
- **name**：密钥对名称，通过引用输入变量keypair_name进行赋值
- **key_file**：私钥文件路径，通过引用输入变量private_key_path进行赋值

### 4. 创建安全组规则

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建安全组规则资源：

```hcl
variable "security_group_name" {
  description = "安全组名称"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组规则资源
resource "huaweicloud_networking_secgroup_rule" "test" {
  count = var.security_group_name != "" ? 1 : 0

  security_group_id = huaweicloud_networking_secgroup.test[0].id
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = "22"
  port_range_max    = "22"
  remote_ip_prefix  = "0.0.0.0/0"
}
```

**参数说明**：
- **count**：资源的创建数，用于控制是否创建安全组规则资源，仅当 `var.security_group_name` 不为空时创建资源
- **security_group_id**：安全组ID，通过引用安全组资源（huaweicloud_networking_secgroup.test[0]）的ID进行赋值
- **direction**：方向，设置为"ingress"表示入站规则
- **ethertype**：以太网类型，设置为"IPv4"表示IPv4协议
- **protocol**：协议，设置为"tcp"表示TCP协议
- **port_range_min**：端口范围最小值，设置为"22"表示SSH端口
- **port_range_max**：端口范围最大值，设置为"22"表示SSH端口
- **remote_ip_prefix**：远程IP前缀，设置为"0.0.0.0/0"表示允许所有IP访问

### 5. 创建EIP

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

### 6. 创建ECS实例

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建ECS实例资源：

```hcl
variable "instance_name" {
  description = "ECS实例名称"
  type        = string
}

variable "instance_user_data" {
  description = "ECS实例用户数据脚本"
  type        = string
  default     = ""
}

variable "instance_system_disk_type" {
  description = "ECS实例系统盘类型"
  type        = string
  default     = "SSD"
}

variable "instance_system_disk_size" {
  description = "ECS实例系统盘大小（GB）"
  type        = number
  default     = 40
}

variable "security_group_ids" {
  description = "ECS实例安全组ID列表"
  type        = list(string)
  default     = []
  nullable    = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ECS实例资源
resource "huaweicloud_compute_instance" "test" {
  name               = var.instance_name
  image_id           = var.instance_image_id == "" ? try(data.huaweicloud_images_images.test[0].images[0].id, null) : var.instance_image_id
  flavor_id          = var.instance_flavor_id == "" ? try(data.huaweicloud_compute_flavors.test[0].flavors[0].id, null) : var.instance_flavor_id
  availability_zone  = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
  key_pair           = huaweicloud_kps_keypair.test.name
  user_data          = var.instance_user_data
  system_disk_type   = var.instance_system_disk_type
  system_disk_size   = var.instance_system_disk_size
  security_group_ids = length(var.security_group_ids) == 0 ? huaweicloud_networking_secgroup.test[*].id : var.security_group_ids

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }

  lifecycle {
    precondition {
      condition     = length(var.security_group_ids) != 0 || var.security_group_name != ""
      error_message = "The security_group_ids must be a non-empty list if the security_group_name is not provided."
    }
  }
}
```

**参数说明**：
- **name**：实例名称，通过引用输入变量instance_name进行赋值
- **image_id**：镜像ID，优先使用输入变量，如果为空则使用镜像列表查询数据源的第一个结果
- **flavor_id**：规格ID，优先使用输入变量，如果为空则使用ECS规格列表查询数据源的第一个结果
- **availability_zone**：可用区，优先使用输入变量，如果为空则使用可用区列表查询数据源的第一个结果
- **key_pair**：密钥对名称，通过引用密钥对资源（huaweicloud_kps_keypair.test）的名称进行赋值
- **user_data**：用户数据脚本，通过引用输入变量instance_user_data进行赋值
- **system_disk_type**：系统盘类型，通过引用输入变量instance_system_disk_type进行赋值
- **system_disk_size**：系统盘大小，通过引用输入变量instance_system_disk_size进行赋值
- **security_group_ids**：安全组ID列表，优先使用输入变量，如果为空则使用安全组资源列表
- **network.uuid**：网络UUID，通过引用VPC子网资源（huaweicloud_vpc_subnet.test）的ID进行赋值
- **lifecycle.precondition**：生命周期前置条件，确保安全组配置正确

### 7. 绑定EIP

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建EIP绑定资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建EIP绑定资源
resource "huaweicloud_compute_eip_associate" "test" {
  public_ip   = var.associate_eip_address == "" ? huaweicloud_vpc_eip.test[0].address : var.associate_eip_address
  instance_id = huaweicloud_compute_instance.test.id
}
```

**参数说明**：
- **public_ip**：公网IP地址，优先使用输入变量，如果为空则使用EIP资源的地址
- **instance_id**：实例ID，通过引用ECS实例资源（huaweicloud_compute_instance.test）的ID进行赋值

### 8. 创建provisioner远程执行

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建provisioner远程执行资源：

```hcl
variable "instance_remote_exec_inline" {
  description = "远程执行的内联脚本"
  type        = list(string)
  nullable    = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建provisioner远程执行资源
resource "null_resource" "test" {
  depends_on = [huaweicloud_compute_eip_associate.test]

  provisioner "remote-exec" {
    connection {
      user        = "root"
      private_key = file(var.private_key_path)
      host        = huaweicloud_compute_eip_associate.test.public_ip
    }

    inline = var.instance_remote_exec_inline
  }
}
```

**参数说明**：
- **depends_on**：依赖关系，确保在EIP绑定完成后执行
- **provisioner.remote-exec**：远程执行provisioner
- **connection.user**：连接用户，设置为"root"
- **connection.private_key**：私钥文件，通过引用输入变量private_key_path进行赋值
- **connection.host**：连接主机，使用EIP绑定资源的公网IP
- **inline**：内联脚本，通过引用输入变量instance_remote_exec_inline进行赋值

### 9. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 网络配置
vpc_name    = "tf_test-vpc"
subnet_name = "tf_test-subnet"

# 安全组配置
security_group_name = "tf_test-security-group"

# 密钥对配置
keypair_name     = "tf_test-keypair"
private_key_path = "./id_rsa"

# EIP配置
bandwidth_name = "tf_test_for_instance"

# ECS实例配置
instance_name = "tf_test_instance_provisioner"
instance_user_data = <<EOF
#!/bin/bash
echo "Hello, World!" > /home/test.txt
EOF

# 远程执行配置
instance_remote_exec_inline = [
  "cat /home/test.txt"
]
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
3. 确认资源计划无误后，运行 `terraform apply` 开始创建通过provisioner远程登录的ECS实例
4. 运行 `terraform show` 查看已创建的通过provisioner远程登录的ECS实例

## 参考信息

- [华为云ECS产品文档](https://support.huaweicloud.com/ecs/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [ECS最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ecs)
