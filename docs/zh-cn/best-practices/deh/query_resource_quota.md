# 部署查询资源配额

## 应用场景

专属主机（Dedicated Host，DEH）资源配额查询是DEH服务提供的配额查询功能，用于查询专属主机相关的资源配额信息。通过查询资源配额，您可以了解当前账户下专属主机资源的配额使用情况，包括已使用的配额、可用的配额和已耗尽的配额，帮助您合理规划资源使用，避免因配额不足导致资源创建失败。通过Terraform自动化查询DEH资源配额，可以确保配额查询的规范化和一致性，提高运维效率。本最佳实践将介绍如何使用Terraform自动化查询DEH资源配额。

## 相关资源/数据源

本最佳实践涉及以下主要数据源：

### 数据源

- [DEH配额数据源（huaweicloud_deh_quotas）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/deh_quotas)

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 查询DEH配额数据源

在TF文件（如main.tf）中添加以下脚本以告知Terraform查询DEH配额信息：

```hcl
variable "host_type" {
  description = "The type of the dedicated host to filter quotas"
  type        = string
  default     = ""
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询DEH配额数据源
data "huaweicloud_deh_quotas" "test" {
  resource = var.host_type != "" ? var.host_type : null
}

# 查询已使用的配额
locals {
  quotas_with_usage = [for v in data.huaweicloud_deh_quotas.test.quota_set : v if v.used > 0]

  # 查询可用的配额
  quotas_available = [for v in data.huaweicloud_deh_quotas.test.quota_set : v if v.hard_limit > v.used]

  # 查询已耗尽的配额
  quotas_exhausted = [for v in data.huaweicloud_deh_quotas.test.quota_set : v if v.hard_limit == v.used]
}
```

**参数说明**：
- **resource**：专属主机类型，通过引用输入变量host_type进行赋值，可选参数，默认值为null（查询所有配额）
- **quota_set**：配额集合，包含以下字段：
  - **type**：配额类型
  - **used**：已使用的配额数量
  - **hard_limit**：配额上限
  - **unit**：配额单位

**本地值说明**：
- **quotas_with_usage**：已使用的配额列表，用于识别哪些资源正在使用
- **quotas_available**：可用的配额列表，用于识别哪些资源仍可创建
- **quotas_exhausted**：已耗尽的配额列表，用于识别哪些资源需要增加配额

### 3. 预设资源部署所需的入参（可选）

本实践中，部分数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 专属主机类型配置（可选，用于过滤特定主机类型的配额）
host_type = "s6"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
   - 如果`host_type`为空字符串或未设置，将查询所有主机类型的配额
   - 如果`host_type`设置为特定值（如"s6"），将只查询该主机类型的配额
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="host_type=s6"`
2. 环境变量：`export TF_VAR_host_type=s6`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来查询DEH资源配额：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看数据源查询计划
3. 确认查询计划无误后，运行 `terraform apply` 开始查询配额信息
4. 运行 `terraform output` 查看配额查询结果

**输出说明**：

查询完成后，可以通过以下方式查看配额信息：

- 查看已使用的配额：`terraform output quotas_with_usage`
- 查看可用的配额：`terraform output quotas_available`
- 查看已耗尽的配额：`terraform output quotas_exhausted`

> 注意：配额查询可以帮助您了解当前账户下专属主机资源的配额使用情况。通过查询已使用的配额，可以识别哪些资源正在使用；通过查询可用的配额，可以识别哪些资源仍可创建；通过查询已耗尽的配额，可以识别哪些资源需要增加配额。如果host_type参数为空，将查询所有主机类型的配额信息。

## 参考信息

- [华为云DEH产品文档](https://support.huaweicloud.com/deh/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [查询资源配额最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/deh/query-resource-quota)
