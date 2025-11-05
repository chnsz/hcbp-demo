# 部署托管连接

## 应用场景

云专线（Direct Connect, DC）是华为云提供的高性能、低延迟、安全可靠的专线接入服务，为企业提供从本地数据中心到华为云的专用网络连接。云专线服务支持多种接入方式，包括物理专线、虚拟专线等，满足不同规模和场景的网络连接需求。

托管连接是云专线服务中的一种连接类型，允许在现有的运营连接上创建虚拟连接，实现多租户共享物理专线资源。通过托管连接，可以降低专线成本，提高资源利用率，满足中小企业和多租户场景的网络连接需求。本最佳实践将介绍如何使用Terraform自动化部署DC托管连接，包括连接创建、带宽配置和VLAN分配。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

无

### 资源

- [DC托管连接资源（huaweicloud_dc_hosted_connect）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dc_hosted_connect)

### 资源/数据源依赖关系

```
huaweicloud_dc_hosted_connect.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建DC托管连接

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建DC托管连接资源：

```hcl
variable "hosted_connect_name" {
  description = "托管连接名称"
  type        = string
}

variable "hosted_connect_description" {
  description = "托管连接描述"
  type        = string
  default     = "Created by Terraform"
}

variable "bandwidth" {
  description = "托管连接带宽大小（Mbit/s）"
  type        = number
}

variable "hosting_id" {
  description = "托管连接所基于的运营连接ID"
  type        = string
}

variable "vlan" {
  description = "分配给托管连接的VLAN"
  type        = number
}

variable "resource_tenant_id" {
  description = "托管连接所属租户ID"
  type        = string
}

variable "peer_location" {
  description = "连接另一端本地设施的位置"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DC托管连接资源
resource "huaweicloud_dc_hosted_connect" "test" {
  name               = var.hosted_connect_name
  description        = var.hosted_connect_description
  bandwidth          = var.bandwidth
  hosting_id         = var.hosting_id
  vlan               = var.vlan
  resource_tenant_id = var.resource_tenant_id
  peer_location      = var.peer_location
}
```

**参数说明**：
- **name**：托管连接名称，通过引用输入变量hosted_connect_name进行赋值
- **description**：托管连接描述，通过引用输入变量hosted_connect_description进行赋值
- **bandwidth**：带宽大小，通过引用输入变量bandwidth进行赋值，单位为Mbit/s
- **hosting_id**：运营连接ID，通过引用输入变量hosting_id进行赋值，指定托管连接所基于的运营连接
- **vlan**：VLAN ID，通过引用输入变量vlan进行赋值，用于网络隔离
- **resource_tenant_id**：租户ID，通过引用输入变量resource_tenant_id进行赋值，指定托管连接所属的租户
- **peer_location**：对端位置，通过引用输入变量peer_location进行赋值，描述连接另一端本地设施的位置

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 托管连接配置
hosted_connect_name        = "tf_test_hosted_connect"
hosted_connect_description = "Created by Terraform"
bandwidth                  = 100
hosting_id                 = "tf_test_hosting_id"
vlan                       = 100
resource_tenant_id         = "tf_test_resource_tenant_id"
peer_location              = "Beijing Data Center"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="hosted_connect_name=my-connect" -var="bandwidth=100"`
2. 环境变量：`export TF_VAR_hosted_connect_name=my-connect`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DC托管连接
4. 运行 `terraform show` 查看已创建的DC托管连接

## 参考信息

- [华为云DC产品文档](https://support.huaweicloud.com/dc/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DC托管连接最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dc/hosted_connect)
