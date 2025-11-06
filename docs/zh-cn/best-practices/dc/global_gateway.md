# 部署全域接入网关

## 应用场景

云专线（Direct Connect, DC）是华为云提供的高性能、低延迟、安全可靠的专线接入服务，为企业提供从本地数据中心到华为云的专用网络连接。云专线服务支持多种接入方式，包括物理专线、虚拟专线等，满足不同规模和场景的网络连接需求。

全域接入网关是云专线服务中的高级组件，用于建立跨地域、跨网络的全局专线连接。通过全域接入网关，可以实现多地域、多网络的统一接入管理，支持BGP路由协议，满足大型企业和复杂网络环境的需求。本最佳实践将介绍如何使用Terraform自动化部署DC全域接入网关，包括网关创建、BGP配置和标签管理。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

无

### 资源

- [DC全域接入网关资源（huaweicloud_dc_global_gateway）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dc_global_gateway)

### 资源/数据源依赖关系

```
huaweicloud_dc_global_gateway.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建DC全域接入网关

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建DC全域接入网关资源：

```hcl
variable "global_gateway_name" {
  description = "全域接入网关名称"
  type        = string
}

variable "global_gateway_description" {
  description = "全域接入网关描述"
  type        = string
  default     = "Created by Terraform"
}

variable "address_family" {
  description = "全域接入网关IP地址族"
  type        = string
  default     = "ipv4"
}

variable "bgp_asn" {
  description = "全域接入网关BGP ASN"
  type        = number
}

variable "enterprise_project_id" {
  description = "全域接入网关所属企业项目ID"
  type        = string
  default     = "0"
}

variable "global_gateway_tags" {
  description = "全域接入网关标签"
  type        = map(string)
  default = {
    "Owner" = "terraform"
    "Env"   = "test"
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DC全域接入网关资源
resource "huaweicloud_dc_global_gateway" "test" {
  name                  = var.global_gateway_name
  description           = var.global_gateway_description
  address_family        = var.address_family
  bgp_asn               = var.bgp_asn
  enterprise_project_id = var.enterprise_project_id

  tags = var.global_gateway_tags
}
```

**参数说明**：
- **name**：全域接入网关名称，通过引用输入变量global_gateway_name进行赋值
- **description**：全域接入网关描述，通过引用输入变量global_gateway_description进行赋值
- **address_family**：IP地址族，通过引用输入变量address_family进行赋值，支持"ipv4"和"ipv6"
- **bgp_asn**：BGP ASN，通过引用输入变量bgp_asn进行赋值，用于BGP路由协议配置
- **enterprise_project_id**：企业项目ID，通过引用输入变量enterprise_project_id进行赋值
- **tags**：标签，通过引用输入变量global_gateway_tags进行赋值，用于资源分类和管理

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 全域接入网关配置
global_gateway_name        = "tf_test_global_gateway"
global_gateway_description = "Created by Terraform"
address_family             = "ipv4"
bgp_asn                    = 65000
enterprise_project_id      = "0"

# 标签配置
global_gateway_tags = {
  "Owner"     = "terraform"
  "Env"       = "test"
  "Project"   = "dc-demo"
  "CostCenter" = "IT"
}
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="global_gateway_name=my-gateway" -var="bgp_asn=65000"`
2. 环境变量：`export TF_VAR_global_gateway_name=my-gateway`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DC全域接入网关
4. 运行 `terraform show` 查看已创建的DC全域接入网关

## 参考信息

- [华为云DC产品文档](https://support.huaweicloud.com/dc/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DC全域接入网关最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dc/global-gateway)
