# 部署自定义字符掩码算法

## 应用场景

数据安全中心（Data Security Center，DSC）是华为云提供的一站式数据安全治理服务，支持敏感数据识别、数据脱敏、数据水印等能力，帮助企业满足数据安全合规要求。在数据脱敏场景中，掩码算法用于对敏感字段进行脱敏处理，保证数据在开发、测试、共享等环节中的安全性。

本最佳实践将介绍如何使用Terraform自动化部署DSC自定义字符掩码算法，通过字符覆盖组合（`PRESNM` 与 `MASK_BY_OVERWRITE`）保留源数据首尾指定数量的字符，并使用替换字符对中间部分进行掩码处理。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [DSC掩码算法（huaweicloud_dsc_mask_algorithm）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dsc_mask_algorithm)

### 资源/数据源依赖关系

```
huaweicloud_dsc_mask_algorithm
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建DSC自定义字符掩码算法

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DSC自定义字符掩码算法资源
variable "mask_algorithm_name" {
  description = "The name of the mask algorithm"
  type        = string
}

variable "mask_algorithm_prefix_length" {
  description = "The number of characters to retain at the beginning of the data"
  type        = number
  default     = 6
}

variable "mask_algorithm_suffix_length" {
  description = "The number of characters to retain at the end of the data"
  type        = number
  default     = 4
}

variable "mask_algorithm_replacement" {
  description = "The character used to mask the data"
  type        = string
  default     = "*"
}

resource "huaweicloud_dsc_mask_algorithm" "test" {
  algorithm_name = var.mask_algorithm_name
  algorithm      = "PRESNM"
  algorithm_type = "MASK_BY_OVERWRITE"
  category       = "BUILT_SELF"

  parameter = jsonencode({
    type   = "CHAR"
    first  = var.mask_algorithm_prefix_length
    second = var.mask_algorithm_suffix_length
    method = var.mask_algorithm_replacement
  })
}
```

**参数说明**：
- **algorithm_name**：掩码算法名称，通过引用输入变量 mask_algorithm_name 进行赋值，名称不能与当前DSC实例中已有的掩码算法名称重复
- **algorithm**：算法类型，固定为 `PRESNM`，表示保留首尾字符的掩码算法
- **algorithm_type**：算法处理方式，固定为 `MASK_BY_OVERWRITE`，表示使用覆盖方式对数据进行掩码
- **category**：算法分类，固定为 `BUILT_SELF`，表示自定义算法
- **parameter**：算法参数，以JSON字符串形式传入，包含以下字段：
  - **type**：参数类型，固定为 `CHAR`，表示字符类型
  - **first**：保留源数据开头字符的数量，通过引用输入变量 mask_algorithm_prefix_length 进行赋值，默认为6
  - **second**：保留源数据结尾字符的数量，通过引用输入变量 mask_algorithm_suffix_length 进行赋值，默认为4
  - **method**：用于掩码的替换字符，通过引用输入变量 mask_algorithm_replacement 进行赋值，默认为 `*`

### 3. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 资源变量
mask_algorithm_name = "tfmaskalgorithm"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="mask_algorithm_name=tfmaskalgorithm"`
2. 环境变量：`export TF_VAR_mask_algorithm_name=tfmaskalgorithm`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 4. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DSC自定义字符掩码算法
4. 运行 `terraform show` 查看已创建的DSC自定义字符掩码算法

## 参考信息

- [华为云数据安全中心产品文档](https://support.huaweicloud.com/dsc/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DSC自定义字符掩码算法最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dsc/custom-character-mask-algorithm)
