# 部署全球连接带宽

## 应用场景

云连接（Cloud Connect，CC）全球连接带宽是云连接服务提供的全球网络带宽资源，用于为跨区域、跨国家的网络连接提供带宽保障。通过创建全球连接带宽，您可以统一管理和分配全球范围内的网络带宽资源，实现灵活的网络带宽配置和跨境网络连接。通过Terraform自动化创建全球连接带宽，可以确保带宽资源管理的规范化和一致性，提高运维效率。本最佳实践将介绍如何使用Terraform自动化创建云连接全球连接带宽。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [云连接全球连接带宽资源（huaweicloud_cc_global_connection_bandwidth）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cc_global_connection_bandwidth)

### 资源/数据源依赖关系

```
huaweicloud_cc_global_connection_bandwidth
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建云连接全球连接带宽资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建云连接全球连接带宽资源：

```hcl
variable "global_connection_bandwidth_name" {
  description = "The name of the global connection bandwidth"
  type        = string
}

variable "bandwidth_type" {
  description = "The type of the global connection bandwidth"
  type        = string
}

variable "bordercross" {
  description = "Whether the global connection bandwidth crosses borders"
  type        = bool
}

variable "bandwidth_size" {
  description = "Bandwidth size of the global connection bandwidth"
  type        = number
}

variable "charge_mode" {
  description = "Billing option of the global connection bandwidth"
  type        = string
}

variable "global_connection_bandwidth_description" {
  description = "The description about the global connection bandwidth"
  type        = string
  default     = "Created by Terraform"
}

variable "global_connection_bandwidth_tags" {
  description = "The tags of the global connection bandwidth"
  type        = map(string)
  default     = {
    "Owner" = "terraform"
    "Env"   = "test"
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云连接全球连接带宽资源
resource "huaweicloud_cc_global_connection_bandwidth" "test" {
  name        = var.global_connection_bandwidth_name
  type        = var.bandwidth_type
  bordercross = var.bordercross
  size        = var.bandwidth_size
  charge_mode = var.charge_mode
  description = var.global_connection_bandwidth_description

  tags = var.global_connection_bandwidth_tags
}
```

**参数说明**：
- **name**：全球连接带宽名称，通过引用输入变量global_connection_bandwidth_name进行赋值
- **type**：全球连接带宽类型，通过引用输入变量bandwidth_type进行赋值
- **bordercross**：全球连接带宽是否跨境，通过引用输入变量bordercross进行赋值
- **size**：全球连接带宽的带宽大小，通过引用输入变量bandwidth_size进行赋值
- **charge_mode**：全球连接带宽的计费方式，通过引用输入变量charge_mode进行赋值
- **description**：全球连接带宽的描述信息，通过引用输入变量global_connection_bandwidth_description进行赋值，默认值为"Created by Terraform"
- **tags**：全球连接带宽的标签，通过引用输入变量global_connection_bandwidth_tags进行赋值，默认值包含"Owner"和"Env"标签

### 3. 预设资源部署所需的入参（可选）

本实践中，资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 云连接全球连接带宽配置
global_connection_bandwidth_name = "tf_test_global_connection_bandwidth"
bandwidth_type                   = "bgp"
bordercross                      = true
bandwidth_size                   = 5
charge_mode                      = "bandwidth"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="global_connection_bandwidth_name=test-bandwidth" -var="bandwidth_size=10"`
2. 环境变量：`export TF_VAR_global_connection_bandwidth_name=test-bandwidth` 和 `export TF_VAR_bandwidth_size=10`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建云连接全球连接带宽：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建云连接全球连接带宽
4. 运行 `terraform show` 查看已创建的云连接全球连接带宽详情

## 参考信息

- [华为云云连接产品文档](https://support.huaweicloud.com/cc/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [全球连接带宽最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cc/global-connection-bandwidth)
