# 部署用户

## 应用场景

虚拟专用网络（Virtual Private Network，VPN）是华为云提供的安全、可靠的网络连接服务，支持在VPC与本地网络之间建立加密的IPsec VPN连接，实现云上资源与本地数据中心的互联互通。VPN用户是VPN服务的重要功能，用于创建和管理VPN服务器上的用户账号，实现点到站点VPN连接的用户认证。通过创建VPN用户，可以为不同的用户分配独立的账号和密码，实现用户级别的访问控制和权限管理。本最佳实践将介绍如何使用Terraform自动化部署VPN用户，包括VPN用户的创建和配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [VPN用户资源（huaweicloud_vpn_user）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpn_user)

### 资源/数据源依赖关系

```text
huaweicloud_vpn_user
```

> 注意：VPN用户需要关联VPN服务器。在创建VPN用户之前，需要确保VPN服务器已经创建完成。

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建VPN用户

在TF文件（如main.tf）中添加以下脚本以创建VPN用户：

```hcl
# 创建VPN用户资源
resource "huaweicloud_vpn_user" "test" {
  vpn_server_id = var.vpn_user_server_id
  name          = var.vpn_user_name
  password      = var.vpn_user_password
}
```

**参数说明**：
- **vpn_server_id**：VPN服务器ID，通过引用输入变量 `vpn_user_server_id` 进行赋值，用于指定VPN用户所属的VPN服务器
- **name**：VPN用户名称，通过引用输入变量 `vpn_user_name` 进行赋值
- **password**：VPN用户密码，通过引用输入变量 `vpn_user_password` 进行赋值，用于用户认证

> 注意：VPN用户用于点到站点VPN连接的用户认证。创建VPN用户后，用户可以使用该账号和密码连接到VPN服务器，实现安全的远程访问。

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# VPN用户配置（必填）
vpn_user_server_id = "your-vpn-server-id"
vpn_user_name      = "tf_test_vpn_user"
vpn_user_password  = "your-secure-password"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpn_user_server_id=your-vpn-server-id" -var="vpn_user_name=tf_test_vpn_user"`
2. 环境变量：`export TF_VAR_vpn_user_server_id=your-vpn-server-id`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建VPN用户
4. 运行 `terraform show` 查看已创建的VPN用户

## 参考信息

- [华为云VPN产品文档](https://support.huaweicloud.com/vpn/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [VPN用户最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/vpn/user)
