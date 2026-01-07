# 部署修改实例类型

## 应用场景

云堡垒机（Cloud Bastion Host，CBH）实例类型修改是CBH服务提供的重要运维功能，当您需要调整云堡垒机实例的计算能力、存储空间或网络性能时，可以通过修改实例类型来满足业务需求的变化。通过Terraform自动化执行实例类型修改操作，可以确保资源配置的规范化和一致性，提高运维效率。本最佳实践将介绍如何使用Terraform自动化修改单节点CBH实例的类型。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [云堡垒机修改实例类型资源（huaweicloud_cbh_change_instance_type）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cbh_change_instance_type)

### 资源/数据源依赖关系

```
huaweicloud_cbh_change_instance_type
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建云堡垒机修改实例类型资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建云堡垒机修改实例类型资源：

```hcl
variable "server_id" {
  description = "The ID of the single node CBH instance to change type"
  type        = string
}

variable "availability_zone" {
  description = "The availability zone of the single-node CBH instance to change type"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云堡垒机修改实例类型资源
resource "huaweicloud_cbh_change_instance_type" "test" {
  server_id         = var.server_id
  availability_zone = var.availability_zone
}
```

**参数说明**：
- **server_id**：要修改类型的单节点CBH实例ID，通过引用输入变量server_id进行赋值
- **availability_zone**：单节点CBH实例的可用区，通过引用输入变量availability_zone进行赋值，默认值为空字符串

### 3. 预设资源部署所需的入参（可选）

本实践中，资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 云堡垒机修改实例类型配置
server_id = "your_single_node_server_id"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="server_id=your-server-id" -var="availability_zone=your-az"`
2. 环境变量：`export TF_VAR_server_id=your-server-id` 和 `export TF_VAR_availability_zone=your-az`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来修改单节点CBH实例类型：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始修改单节点CBH实例类型
4. 运行 `terraform show` 查看已创建的云堡垒机修改实例类型资源详情

> 注意：修改单节点CBH实例类型大约需要15分钟，请耐心等待。

## 参考信息

- [华为云云堡垒机产品文档](https://support.huaweicloud.com/cbh/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [修改实例类型最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cbh/change-instance-type)
