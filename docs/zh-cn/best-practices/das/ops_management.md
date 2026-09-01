# 部署运维管理

## 应用场景

数据管理服务（Data Admin Service，简称DAS）提供面向DBA的智能运维能力，支持对数据库实例进行分组管理，并通过邮件模板订阅巡检报告通知，帮助用户批量管理实例健康状态并及时获取诊断结果。通过创建实例组、分配实例，以及配置邮件模板与批量订阅，可实现巡检报告的自动化通知与统一运维。本最佳实践将介绍如何使用Terraform自动化部署DAS运维管理相关配置，包括创建实例组、分配实例到实例组、创建邮件模板，以及批量订阅邮件模板。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 资源

- [实例组资源（huaweicloud_das_instance_group）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_instance_group)
- [实例组分配资源（huaweicloud_das_instance_group_assign）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_instance_group_assign)
- [邮件模板资源（huaweicloud_das_email_template）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_email_template)
- [邮件模板批量操作资源（huaweicloud_das_email_templates_batch_action）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/das_email_templates_batch_action)

### 资源/数据源依赖关系

```text
huaweicloud_das_instance_group
    ├── huaweicloud_das_instance_group_assign
    └── huaweicloud_das_email_template
        └── huaweicloud_das_email_templates_batch_action
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建实例组

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建实例组资源：

```hcl
variable "ops_datastore_type" {
  description = "The database type"
  type        = string
}

variable "ops_group_name" {
  description = "The instance group name"
  type        = string
}

variable "ops_group_description" {
  description = "The description of the instance group"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建实例组资源
resource "huaweicloud_das_instance_group" "test" {
  datastore_type = var.ops_datastore_type
  group_name     = var.ops_group_name
  description    = var.ops_group_description
}
```

**参数说明**：
- **datastore_type**：数据库类型，通过引用输入变量 `ops_datastore_type` 进行赋值
- **group_name**：实例组名称，通过引用输入变量 `ops_group_name` 进行赋值
- **description**：实例组描述信息，通过引用输入变量 `ops_group_description` 进行赋值

### 3. 分配实例到实例组

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建实例组分配资源：

```hcl
variable "ops_group_instance_ids" {
  description = "The list of instance IDs to be assigned to the group"
  type        = list(string)
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建实例组分配资源
resource "huaweicloud_das_instance_group_assign" "test" {
  group_id     = huaweicloud_das_instance_group.test.id
  instance_ids = var.ops_group_instance_ids
}
```

**参数说明**：
- **group_id**：实例组ID，通过引用实例组资源的ID进行赋值
- **instance_ids**：待分配到实例组的实例ID列表，通过引用输入变量 `ops_group_instance_ids` 进行赋值

> 注意：分配前需确保目标数据库实例已存在，且实例类型与实例组的数据库类型匹配。

### 4. 创建邮件模板

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建邮件模板资源：

```hcl
variable "ops_email_template_name" {
  description = "The name of the email template"
  type        = string
}

variable "ops_email_health_rank" {
  description = "The list of health ranks"
  type        = list(string)
}

variable "ops_email_inspection_time" {
  description = "The diagnosis time"
  type        = string
}

variable "ops_email_send_time" {
  description = "The send time"
  type        = string
}

variable "ops_email_time_zone" {
  description = "The time zone"
  type        = string
}

variable "ops_email_address" {
  description = "The email address for notification"
  type        = string
  default     = null
}

variable "ops_email_topic" {
  description = "The topic ID for notification"
  type        = string
  default     = null
}

variable "ops_email_topic_urn" {
  description = "The topic URN for notification"
  type        = string
  default     = null
}

variable "ops_email_obs_bucket_name" {
  description = "The OBS bucket name for storing inspection reports"
  type        = string
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建邮件模板资源
resource "huaweicloud_das_email_template" "test" {
  datastore_type  = var.ops_datastore_type
  name            = var.ops_email_template_name
  groups          = [huaweicloud_das_instance_group.test.id]
  health_rank     = var.ops_email_health_rank
  inspection_time = var.ops_email_inspection_time
  send_time       = var.ops_email_send_time
  time_zone       = var.ops_email_time_zone
  email           = var.ops_email_address
  topic           = var.ops_email_topic
  topic_urn       = var.ops_email_topic_urn
  obs_bucket_name = var.ops_email_obs_bucket_name
}
```

**参数说明**：
- **datastore_type**：数据库类型，通过引用输入变量 `ops_datastore_type` 进行赋值
- **name**：邮件模板名称，通过引用输入变量 `ops_email_template_name` 进行赋值
- **groups**：关联的实例组ID列表，通过引用实例组资源的ID进行赋值
- **health_rank**：健康等级列表，通过引用输入变量 `ops_email_health_rank` 进行赋值
- **inspection_time**：诊断时间，通过引用输入变量 `ops_email_inspection_time` 进行赋值
- **send_time**：发送时间，通过引用输入变量 `ops_email_send_time` 进行赋值
- **time_zone**：时区，通过引用输入变量 `ops_email_time_zone` 进行赋值
- **email**：通知邮箱地址，通过引用输入变量 `ops_email_address` 进行赋值
- **topic**：通知主题ID，通过引用输入变量 `ops_email_topic` 进行赋值
- **topic_urn**：通知主题URN，通过引用输入变量 `ops_email_topic_urn` 进行赋值
- **obs_bucket_name**：用于存储巡检报告的OBS桶名称，通过引用输入变量 `ops_email_obs_bucket_name` 进行赋值

> 注意：邮件模板依赖已创建的实例组。请按需配置邮箱、SMN主题或OBS桶等通知与存储参数。

### 5. 批量订阅邮件模板

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建邮件模板批量操作资源：

```hcl
variable "ops_email_subscribe" {
  description = "Whether to subscribe to the email templates"
  type        = bool
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建邮件模板批量操作资源
resource "huaweicloud_das_email_templates_batch_action" "test" {
  subscribe          = var.ops_email_subscribe
  email_template_ids = [huaweicloud_das_email_template.test.id]
}
```

**参数说明**：
- **subscribe**：是否订阅邮件模板，通过引用输入变量 `ops_email_subscribe` 进行赋值
- **email_template_ids**：待批量操作的邮件模板ID列表，通过引用邮件模板资源的ID进行赋值

> 注意：批量订阅依赖已创建的邮件模板。

### 6. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 实例组配置
ops_datastore_type    = "MySQL"
ops_group_name        = "tf_test_das_group"
ops_group_description = "tf_test_das_group_description"
ops_group_instance_ids = ["your_instance_id_1", "your_instance_id_2"]

# 邮件模板配置
ops_email_template_name   = "tf_test_email_template"
ops_email_health_rank     = ["dangerous", "sub_healthy"]
ops_email_inspection_time = "00:00-00:00"
ops_email_send_time       = "08:00-10:00"
ops_email_time_zone       = "Asia/Shanghai"
ops_email_address         = "tf_test@example.com"
ops_email_obs_bucket_name = "your bucket name"

# 邮件模板批量订阅配置
ops_email_subscribe = true
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="ops_group_name=tf_test_das_group" -var="ops_email_subscribe=true"`
2. 环境变量：`export TF_VAR_ops_group_name=tf_test_das_group`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 7. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建运维管理相关配置
4. 运行 `terraform show` 查看已创建的运维管理相关配置

## 参考信息

- [华为云DAS产品文档](https://support.huaweicloud.com/das/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DAS运维管理最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/das/ops-management)
