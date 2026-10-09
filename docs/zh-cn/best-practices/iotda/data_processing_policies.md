# 部署数据流转控制与积压策略

## 应用场景

物联网平台设备接入服务（IoT Device Access，IoTDA）是华为云提供的设备接入与管理服务，支持海量设备连接、数据采集与数据流转。在设备数据转发场景中，如果转发目标出现异常或下游处理能力不足，可能导致转发请求量突增或数据大量积压，进而影响租户下其他业务的正常运行。

本最佳实践将介绍如何使用Terraform自动化部署IoTDA的数据流转控制策略与数据积压策略，通过限制租户级数据转发的TPS上限，并对转发数据的积压大小和积压时间进行控制，保障数据转发链路的稳定性与可靠性。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [数据流转控制策略（huaweicloud_iotda_data_flow_control_policy）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/iotda_data_flow_control_policy)
- [数据积压策略（huaweicloud_iotda_data_backlog_policy）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/iotda_data_backlog_policy)

### 资源/数据源依赖关系

```
huaweicloud_iotda_data_flow_control_policy

huaweicloud_iotda_data_backlog_policy
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建数据流转控制策略

在TF文件（如main.tf）中添加以下脚本以创建数据流转控制策略：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建数据流转控制策略资源
variable "flow_control_policy_name" {
  description = "The name of the data flow control policy"
  type        = string
}

variable "flow_control_policy_description" {
  description = "The description of the data flow control policy"
  type        = string
  default     = ""
}

variable "flow_control_policy_limit" {
  description = "The flow control limit of the policy in tps"
  type        = number
}

resource "huaweicloud_iotda_data_flow_control_policy" "test" {
  name        = var.flow_control_policy_name
  description = var.flow_control_policy_description
  scope       = "USER"
  limit       = var.flow_control_policy_limit
}
```

**参数说明**：
- **name**：数据流转控制策略的名称，通过引用输入变量 flow_control_policy_name 进行赋值，名称长度不超过256个字符，仅支持中文、字母、数字及 `_?'#().,&%@!-` 等字符，不允许包含空格
- **description**：数据流转控制策略的描述，通过引用输入变量 flow_control_policy_description 进行赋值，长度不超过256个字符
- **scope**：数据流转控制策略的作用域，本实践中固定为 `USER`，表示租户级流控，此时无需指定 scope_value 参数
- **limit**：数据流转控制策略的流控大小，单位为tps，通过引用输入变量 flow_control_policy_limit 进行赋值，取值范围为1到1,000，默认值为1,000

### 3. 创建数据积压策略

在TF文件（如main.tf）中添加以下脚本以创建数据积压策略：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建数据积压策略资源
variable "backlog_policy_name" {
  description = "The name of the data backlog policy"
  type        = string
}

variable "backlog_policy_description" {
  description = "The description of the data backlog policy"
  type        = string
  default     = ""
}

variable "backlog_policy_size" {
  description = "The size of data backlog in bytes"
  type        = string
}

variable "backlog_policy_time" {
  description = "The data backlog time in seconds"
  type        = string
}

resource "huaweicloud_iotda_data_backlog_policy" "test" {
  name         = var.backlog_policy_name
  description  = var.backlog_policy_description
  backlog_size = var.backlog_policy_size
  backlog_time = var.backlog_policy_time
}
```

**参数说明**：
- **name**：数据积压策略的名称，通过引用输入变量 backlog_policy_name 进行赋值，名称长度不超过256个字符，仅支持中文、字母、数字及 `_?'#().,&%@!-` 等字符，不允许包含空格
- **description**：数据积压策略的描述，通过引用输入变量 backlog_policy_description 进行赋值，长度不超过256个字符
- **backlog_size**：数据积压大小，单位为字节，通过引用输入变量 backlog_policy_size 进行赋值，取值范围为0到1,073,741,823，0表示不积压
- **backlog_time**：数据积压时间，单位为秒，通过引用输入变量 backlog_policy_time 进行赋值，取值范围为0到86,399，0表示不积压；当同时配置积压大小和积压时间时，以先达到阈值的维度为准

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 数据流转控制策略参数
flow_control_policy_name        = "tf_test_iotda_flow_control_policy"
flow_control_policy_description = "Limit-the-data-forwarding-tps-of-the-tenant"
flow_control_policy_limit       = 500

# 数据积压策略参数
backlog_policy_name        = "tf_test_iotda_backlog_policy"
backlog_policy_description = "Control-the-size-and-time-of-forwarded-data-backlog"
backlog_policy_size        = "524288000"
backlog_policy_time        = "3600"

# IoTDA实例的HTTPS应用接入地址
iotda_access_address = "https://779f0f0dd5.st1.iotda-app.cn-north-4.myhuaweicloud.com"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="flow_control_policy_name=my-policy"`
2. 环境变量：`export TF_VAR_flow_control_policy_name=my-policy`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建数据流转控制策略与数据积压策略
4. 运行 `terraform show` 查看已创建的数据流转控制策略与数据积压策略

> 注意：数据流转控制策略与数据积压策略仅支持标准版和企业版IoTDA实例，需要通过provider块的`endpoints.iotda`参数指定实例的HTTPS应用接入地址，且provider版本需为1.71.0或更高版本。

## 参考信息

- [华为云IoTDA产品文档](https://support.huaweicloud.com/iotda/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [IoTDA数据流转控制与积压策略最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/iotda/data-processing-policies)
