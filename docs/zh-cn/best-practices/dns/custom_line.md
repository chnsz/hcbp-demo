# 部署自定义线路

## 应用场景

云解析服务（Domain Name Service, DNS）是华为云提供的高可用、高性能的域名解析服务，支持公网域名解析和私网域名解析。DNS服务提供智能解析、负载均衡、健康检查等功能，帮助用户实现域名的智能调度和故障转移。

自定义线路是DNS服务中的高级功能，允许用户根据特定的IP地址段创建自定义解析线路，实现更精细的流量调度和解析控制。通过自定义线路，企业可以根据用户的地理位置、网络运营商、IP地址段等因素，为不同的用户群体提供不同的解析结果，实现智能解析和负载均衡。本最佳实践将介绍如何使用Terraform自动化部署DNS自定义线路，包括线路创建和IP地址段配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [DNS自定义线路资源（huaweicloud_dns_custom_line）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dns_custom_line)

### 资源/数据源依赖关系

```
无依赖关系
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建DNS自定义线路

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建DNS自定义线路资源：

```hcl
variable "dns_custom_line_name" {
  description = "自定义线路名称"
  type        = string
}

variable "dns_custom_line_ip_segments" {
  description = "IP地址段"
  type        = list(string)
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DNS自定义线路资源
resource "huaweicloud_dns_custom_line" "test" {
  name        = var.dns_custom_line_name
  ip_segments = var.dns_custom_line_ip_segments
}
```

**参数说明**：
- **name**：自定义线路名称，通过引用输入变量dns_custom_line_name进行赋值
- **ip_segments**：IP地址段列表，通过引用输入变量dns_custom_line_ip_segments进行赋值

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# DNS自定义线路配置
dns_custom_line_name        = "your_custom_line_name"
dns_custom_line_ip_segments = ["100.100.100.102-100.100.100.102", "100.100.100.101-100.100.100.101"]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="dns_custom_line_name=my-line" -var="dns_custom_line_ip_segments=[\"192.168.1.1-192.168.1.10\"]"`
2. 环境变量：`export TF_VAR_dns_custom_line_name=my-line`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DNS自定义线路
4. 运行 `terraform show` 查看已创建的DNS自定义线路

## 参考信息

- [华为云DNS产品文档](https://support.huaweicloud.com/dns/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DNS自定义线路最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dns/custom-line)
