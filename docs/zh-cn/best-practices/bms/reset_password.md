# 部署实例密码重置

## 应用场景

裸金属服务器（Bare Metal Server，BMS）实例密码重置是BMS服务提供的重要运维功能，当您忘记实例密码或需要定期更换密码时，可以通过密码重置功能快速恢复或更新实例的登录密码。通过Terraform自动化执行密码重置操作，可以确保密码管理的规范化和安全性，避免手动操作可能带来的安全风险。本最佳实践将介绍如何使用Terraform自动化重置BMS实例的密码。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [BMS实例密码重置资源（huaweicloud_bms_instance_password_reset）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/bms_instance_password_reset)

### 资源/数据源依赖关系

```
huaweicloud_bms_instance_password_reset
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建BMS实例密码重置资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建BMS实例密码重置资源：

```hcl
variable "bms_instance_id" {
  description = "The ID of the BMS instance"
  type        = string
}

variable "bms_instance_new_password" {
  description = "The new password of the BMS instance"
  type        = string
  sensitive   = true
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建BMS实例密码重置资源
resource "huaweicloud_bms_instance_password_reset" "test" {
  server_id    = var.bms_instance_id
  new_password = var.bms_instance_new_password
}
```

**参数说明**：
- **server_id**：BMS实例ID，通过引用输入变量bms_instance_id进行赋值
- **new_password**：BMS实例的新密码，通过引用输入变量bms_instance_new_password进行赋值，该变量被标记为敏感信息

### 3. 预设资源部署所需的入参（可选）

本实践中，资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# BMS实例密码重置配置
bms_instance_id           = "your_bms_instance_id"
bms_instance_new_password = "your_new_password"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="bms_instance_id=your-instance-id" -var="bms_instance_new_password=your-password"`
2. 环境变量：`export TF_VAR_bms_instance_id=your-instance-id` 和 `export TF_VAR_bms_instance_new_password=your-password`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来重置BMS实例密码：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始重置BMS实例密码
4. 运行 `terraform show` 查看已创建的BMS实例密码重置资源详情

> 注意：销毁此模块不会清除相应的请求记录，只会从tf状态文件中移除资源信息。

## 参考信息

- [华为云BMS产品文档](https://support.huaweicloud.com/bms/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [实例密码重置最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/bms/bms-reset-password)
