# 部署服务

## 应用场景

VPC终端节点（VPC Endpoint，VPCEP）是华为云提供的VPC内资源互访服务，支持在VPC内创建终端节点和终端节点服务，实现VPC内资源的私网访问。终端节点服务是VPCEP服务的核心功能，用于将VPC内的服务资源（如ECS、ELB等）发布为终端节点服务，供其他VPC通过终端节点访问。通过终端节点服务，可以实现跨VPC的私网访问，避免通过公网访问，提高访问安全性和稳定性。本最佳实践将介绍如何使用Terraform自动化部署终端节点服务，包括可用区查询、ECS规格查询、镜像查询、VPC、子网、安全组、ECS实例和终端节点服务的创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [ECS规格查询数据源（data.huaweicloud_compute_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/compute_flavors)
- [镜像查询数据源（data.huaweicloud_images_image）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/images_image)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [ECS实例资源（huaweicloud_compute_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [终端节点服务资源（huaweicloud_vpcep_service）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpcep_service)

### 资源/数据源依赖关系

```text
data.huaweicloud_availability_zones
    └── data.huaweicloud_compute_flavors
        └── huaweicloud_compute_instance
            └── huaweicloud_vpcep_service

data.huaweicloud_images_image
    └── huaweicloud_compute_instance
        └── huaweicloud_vpcep_service

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_compute_instance
            └── huaweicloud_vpcep_service

huaweicloud_networking_secgroup
    └── huaweicloud_compute_instance
        └── huaweicloud_vpcep_service
```

> 注意：终端节点服务需要依赖ECS实例，ECS实例需要依赖VPC、子网、安全组、可用区、规格和镜像资源。终端节点服务通过ECS实例的网络端口ID进行关联。

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询可用区

在TF文件（如main.tf）中添加以下脚本以查询可用区：

```hcl
# 查询可用区数据源
data "huaweicloud_availability_zones" "test" {}
```

### 3. 查询ECS规格

在TF文件（如main.tf）中添加以下脚本以查询ECS规格：

```hcl
# 查询ECS规格数据源
data "huaweicloud_compute_flavors" "test" {
  availability_zone = try(data.huaweicloud_availability_zones.test.names[0], null)
  performance_type  = var.instance_flavor_performance_type
  cpu_core_count    = var.instance_flavor_cpu_core_count
  memory_size       = var.instance_flavor_memory_size
}
```

**参数说明**：
- **availability_zone**：可用区名称，通过引用可用区查询数据源的结果进行赋值
- **performance_type**：规格性能类型，通过引用输入变量 `instance_flavor_performance_type` 进行赋值
- **cpu_core_count**：CPU核数，通过引用输入变量 `instance_flavor_cpu_core_count` 进行赋值
- **memory_size**：内存大小，通过引用输入变量 `instance_flavor_memory_size` 进行赋值

### 4. 查询镜像

在TF文件（如main.tf）中添加以下脚本以查询镜像：

```hcl
# 查询镜像数据源
data "huaweicloud_images_image" "test" {
  name        = var.instance_image_name
  most_recent = var.instance_image_most_recent
}
```

**参数说明**：
- **name**：镜像名称，通过引用输入变量 `instance_image_name` 进行赋值
- **most_recent**：是否使用最新镜像，通过引用输入变量 `instance_image_most_recent` 进行赋值

### 5. 创建VPC和子网

在TF文件（如main.tf）中添加以下脚本以创建VPC和子网：

```hcl
# 创建VPC资源
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}

# 创建VPC子网资源
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量 `vpc_name` 进行赋值
- **cidr**：VPC的CIDR地址块，通过引用输入变量 `vpc_cidr` 进行赋值
- **vpc_id**：子网所属的VPC ID，通过引用VPC资源的ID进行赋值
- **cidr**：子网的CIDR地址块，如果输入变量为空则自动计算，否则使用输入变量的值
- **gateway_ip**：子网的网关IP地址，如果输入变量为空则自动计算，否则使用输入变量的值

### 6. 创建安全组

在TF文件（如main.tf）中添加以下脚本以创建安全组：

```hcl
# 创建安全组资源
resource "huaweicloud_networking_secgroup" "test" {
  name = var.security_group_name
}
```

**参数说明**：
- **name**：安全组名称，通过引用输入变量 `security_group_name` 进行赋值

### 7. 创建ECS实例

在TF文件（如main.tf）中添加以下脚本以创建ECS实例：

```hcl
# 创建ECS实例资源
resource "huaweicloud_compute_instance" "test" {
  name               = var.instance_name
  image_id           = data.huaweicloud_images_image.test.id
  flavor_id          = var.instance_flavor_id == "" ? try(data.huaweicloud_compute_flavors.test.ids[0], "") : var.instance_flavor_id
  security_group_ids = [huaweicloud_networking_secgroup.test.id]
  availability_zone  = try(data.huaweicloud_availability_zones.test.names[0], null)

  network {
    uuid = huaweicloud_vpc_subnet.test.id
  }
}
```

**参数说明**：
- **name**：ECS实例名称，通过引用输入变量 `instance_name` 进行赋值
- **image_id**：镜像ID，通过引用镜像查询数据源的ID进行赋值
- **flavor_id**：规格ID，如果输入变量为空则使用规格查询数据源的结果，否则使用输入变量的值
- **security_group_ids**：安全组ID列表，通过引用安全组资源的ID进行赋值
- **availability_zone**：可用区名称，通过引用可用区查询数据源的结果进行赋值
- **network**：网络配置，通过引用子网资源的ID进行赋值

### 8. 创建终端节点服务

在TF文件（如main.tf）中添加以下脚本以创建终端节点服务：

```hcl
# 创建终端节点服务资源
resource "huaweicloud_vpcep_service" "test" {
  name        = var.endpoint_service_name
  server_type = var.endpoint_service_type
  vpc_id      = huaweicloud_vpc.test.id
  port_id     = huaweicloud_compute_instance.test.network[0].port

  dynamic "port_mapping" {
    for_each = var.endpoint_service_port_mapping

    content {
      service_port  = port_mapping.value.service_port
      terminal_port = port_mapping.value.terminal_port
    }
  }
}
```

**参数说明**：
- **name**：终端节点服务名称，通过引用输入变量 `endpoint_service_name` 进行赋值
- **server_type**：服务器类型，通过引用输入变量 `endpoint_service_type` 进行赋值
- **vpc_id**：终端节点服务所属的VPC ID，通过引用VPC资源的ID进行赋值
- **port_id**：端口ID，通过引用ECS实例的网络端口ID进行赋值
- **port_mapping**：端口映射列表，通过动态块 `dynamic "port_mapping"` 根据输入变量 `endpoint_service_port_mapping` 创建端口映射
  - **service_port**：服务端口，通过引用输入变量中的 `service_port` 进行赋值
  - **terminal_port**：终端端口，通过引用输入变量中的 `terminal_port` 进行赋值

> 注意：终端节点服务需要关联ECS实例的网络端口。端口映射用于配置服务端口和终端端口的对应关系，实现端口转发功能。

### 9. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# VPC和子网配置（必填）
vpc_name   = "tf_test_vpc"
vpc_cidr   = "192.168.0.0/16"
subnet_name = "tf_test_subnet"

# 安全组配置（必填）
security_group_name = "tf_test_security_group"

# ECS实例配置（必填）
instance_name = "tf_test_instance"

# 终端节点服务配置（必填）
endpoint_service_name = "tf-test-service"
endpoint_service_port_mapping = [
  {
    service_port  = 8080
    terminal_port = 80
  }
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=tf_test_vpc"`
2. 环境变量：`export TF_VAR_vpc_name=tf_test_vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 10. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建终端节点服务及相关资源
4. 运行 `terraform show` 查看已创建的终端节点服务

## 参考信息

- [华为云VPCEP产品文档](https://support.huaweicloud.com/vpcep/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [终端节点服务最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/vpcep/service)
