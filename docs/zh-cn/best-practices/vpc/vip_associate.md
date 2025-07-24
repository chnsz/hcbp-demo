# 部署VPC VIP关联

## 应用场景

虚拟IP（Virtual IP，VIP）是华为云VPC中用于实现高可用性的一种网络技术，可以将一个虚拟IP地址绑定到多个ECS实例上，实现负载均衡和故障转移。通过VIP关联，可以实现业务的高可用部署，当某个ECS实例出现故障时，流量可以自动切换到其他健康的实例上。本最佳实践将介绍如何使用Terraform自动化部署VIP关联，包括创建ECS实例、VIP资源及其关联配置。

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
- [安全组规则资源（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [ECS实例资源（huaweicloud_compute_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/compute_instance)
- [VIP资源（huaweicloud_networking_vip）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_vip)
- [VIP关联资源（huaweicloud_networking_vip_associate）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_vip_associate)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    ├── data.huaweicloud_compute_flavors
    │   └── data.huaweicloud_images_images
    │       └── huaweicloud_compute_instance
    └── huaweicloud_compute_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        ├── huaweicloud_compute_instance
        └── huaweicloud_networking_vip
            └── huaweicloud_networking_vip_associate

huaweicloud_networking_secgroup
    ├── huaweicloud_networking_secgroup_rule
    └── huaweicloud_compute_instance

huaweicloud_compute_instance
    └── huaweicloud_networking_vip_associate
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 前置资源准备

本最佳实践需要先创建VPC、子网、安全组和ECS实例等前置资源。请按照[部署基础弹性云服务器](../ecs/simple_instance.md)最佳实践中的以下步骤进行准备：

- **步骤2**：通过数据源查询ECS实例资源创建所需的可用区
- **步骤3**：通过数据源查询ECS实例资源创建所需的规格
- **步骤4**：通过数据源查询ECS实例资源创建所需的镜像
- **步骤5**：创建VPC资源
- **步骤6**：创建VPC子网资源
- **步骤7**：创建安全组资源
- **步骤8**：创建ECS实例

完成上述步骤后，继续执行本最佳实践的后续步骤。

### 3. 创建VIP资源

在TF文件中添加以下脚本以告知Terraform创建VIP资源：

```hcl
# 创建VIP资源
resource "huaweicloud_networking_vip" "test" {
  network_id = huaweicloud_vpc_subnet.test.id
}
```

**参数说明**：
- **network_id**：VIP所属网络的ID，引用前面创建的子网资源的ID

### 4. 创建VIP关联资源

在TF文件中添加以下脚本以告知Terraform创建VIP关联资源：

```hcl
# 创建VIP关联资源
resource "huaweicloud_networking_vip_associate" "test" {
  vip_id   = huaweicloud_networking_vip.test.id
  port_ids = try([
    huaweicloud_compute_instance.test.network[0].port
  ], [])
}
```

**参数说明**：
- **vip_id**：VIP的ID，引用前面创建的VIP资源的ID
- **port_ids**：要关联的端口ID列表，使用try函数获取ECS实例的网络端口ID，如果获取失败则使用空列表

### 5. 预设资源部署所需的入参（可选）

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
vpc_cidr = "192.168.0.0/16"

# 子网配置
subnet_name = "tf_test_subnet"

# 安全组配置
security_group_name = "tf_test_security_group"

# ECS实例配置
instance_name          = "tf_test_instance"
administrator_password = "YourPasswordInput!"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc" -var="subnet_name=my-subnet"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建VIP关联
4. 运行 `terraform show` 查看已创建的VIP关联详情

## 参考信息

- [华为云VPC产品文档](https://support.huaweicloud.com/vpc/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [VPC VIP关联最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/vpc)
