# 部署基础防护

## 应用场景

Anti-DDoS（Anti-Distributed Denial of Service）是华为云提供的分布式拒绝服务攻击防护服务，能够有效防护针对公网IP的DDoS攻击，保障业务的稳定运行。Anti-DDoS基础防护为华为云用户提供免费的DDoS攻击防护能力，当检测到DDoS攻击时，系统会自动启动流量清洗，将攻击流量过滤后，仅将正常流量转发给源站服务器。

本最佳实践将介绍如何使用Terraform自动化部署基础防护，包括创建弹性公网IP（EIP）、消息通知服务（SMN）主题和订阅，以及配置Anti-DDoS基础防护。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [弹性公网IP资源（huaweicloud_vpc_eip）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [消息通知服务主题资源（huaweicloud_smn_topic）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_topic)
- [消息通知服务订阅资源（huaweicloud_smn_subscription）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/smn_subscription)
- [Anti-DDoS基础防护资源（huaweicloud_antiddos_basic）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/antiddos_basic)

### 资源/数据源依赖关系

```
huaweicloud_vpc_eip
    └── huaweicloud_antiddos_basic

huaweicloud_smn_topic
    ├── huaweicloud_smn_subscription
    └── huaweicloud_antiddos_basic
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建弹性公网IP资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建弹性公网IP资源：

```hcl
variable "vpc_eip_publicip_type" {
  description = "The EIP type"
  type        = string
}

variable "vpc_eip_bandwidth_share_type" {
  description = "The bandwidth share type"
  type        = string
}

variable "vpc_eip_bandwidth_name" {
  description = "The bandwidth name"
  type        = string
  default     = null
}

variable "vpc_eip_bandwidth_size" {
  description = "The bandwidth size"
  type        = number
  default     = null
}

variable "vpc_eip_bandwidth_charge_mode" {
  description = "The bandwidth charge mode"
  type        = string
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建弹性公网IP资源
resource "huaweicloud_vpc_eip" "test" {
  publicip {
    type = var.vpc_eip_publicip_type
  }

  bandwidth {
    share_type  = var.vpc_eip_bandwidth_share_type
    name        = var.vpc_eip_bandwidth_name
    size        = var.vpc_eip_bandwidth_size
    charge_mode = var.vpc_eip_bandwidth_charge_mode
  }
}
```

**参数说明**：
- **publicip.type**：弹性公网IP的类型，通过引用输入变量vpc_eip_publicip_type进行赋值
- **bandwidth.share_type**：带宽的共享类型，通过引用输入变量vpc_eip_bandwidth_share_type进行赋值
- **bandwidth.name**：带宽名称，通过引用输入变量vpc_eip_bandwidth_name进行赋值，默认值为null
- **bandwidth.size**：带宽大小，通过引用输入变量vpc_eip_bandwidth_size进行赋值，默认值为null
- **bandwidth.charge_mode**：带宽的计费模式，通过引用输入变量vpc_eip_bandwidth_charge_mode进行赋值，默认值为null

### 3. 创建消息通知服务主题资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建消息通知服务主题资源：

```hcl
variable "smn_topic_name" {
  description = "The name of the topic to be created"
  type        = string
}

variable "smn_topic_display_name" {
  description = "The topic display name"
  type        = string
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建消息通知服务主题资源
resource "huaweicloud_smn_topic" "test" {
  name         = var.smn_topic_name
  display_name = var.smn_topic_display_name
}
```

**参数说明**：
- **name**：主题名称，通过引用输入变量smn_topic_name进行赋值
- **display_name**：主题的显示名称，通过引用输入变量smn_topic_display_name进行赋值，默认值为null

### 4. 创建消息通知服务订阅资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建消息通知服务订阅资源：

```hcl
variable "smn_subscription_endpoint" {
  description = "The message endpoint"
  type        = string
}

variable "smn_subscription_protocol" {
  description = "The protocol of the message endpoint"
  type        = string
}

variable "smn_subscription_remark" {
  description = "The remark information"
  type        = string
  default     = null
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建消息通知服务订阅资源
resource "huaweicloud_smn_subscription" "test" {
  topic_urn = huaweicloud_smn_topic.test.id
  endpoint  = var.smn_subscription_endpoint
  protocol  = var.smn_subscription_protocol
  remark    = var.smn_subscription_remark
}
```

**参数说明**：
- **topic_urn**：主题的URN，引用前面创建的消息通知服务主题资源（huaweicloud_smn_topic.test）的ID
- **endpoint**：消息接收终端地址，通过引用输入变量smn_subscription_endpoint进行赋值
- **protocol**：消息接收终端的协议，通过引用输入变量smn_subscription_protocol进行赋值
- **remark**：备注信息，通过引用输入变量smn_subscription_remark进行赋值，默认值为null

### 5. 创建Anti-DDoS基础防护资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建Anti-DDoS基础防护资源：

```hcl
variable "antiddos_traffic_threshold" {
  description = "The traffic cleaning threshold in Mbps"
  type        = number
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建Anti-DDoS基础防护资源
resource "huaweicloud_antiddos_basic" "test" {
  traffic_threshold = var.antiddos_traffic_threshold
  eip_id            = huaweicloud_vpc_eip.test.id
  topic_urn         = huaweicloud_smn_topic.test.id
}
```

**参数说明**：
- **traffic_threshold**：流量清洗阈值（单位：Mbps），通过引用输入变量antiddos_traffic_threshold进行赋值
- **eip_id**：需要防护的弹性公网IP的ID，引用前面创建的弹性公网IP资源（huaweicloud_vpc_eip.test）的ID
- **topic_urn**：消息通知服务主题的URN，引用前面创建的消息通知服务主题资源（huaweicloud_smn_topic.test）的ID

### 6. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 弹性公网IP配置
vpc_eip_publicip_type         = "5_bgp"
vpc_eip_bandwidth_share_type  = "PER"
vpc_eip_bandwidth_name        = "test-antiddos-basic-name"
vpc_eip_bandwidth_size        = 5
vpc_eip_bandwidth_charge_mode = "traffic"

# 消息通知服务主题配置
smn_topic_name         = "test-antiddos-basic-name"
smn_topic_display_name = "The display name of topic test-antiddos-basic-name"

# 消息通知服务订阅配置
smn_subscription_endpoint = "mailtest@gmail.com"
smn_subscription_protocol = "email"
smn_subscription_remark   = "test remark"

# Anti-DDoS基础防护配置
antiddos_traffic_threshold = 200
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_eip_publicip_type=5_bgp" -var="smn_topic_name=test-topic"`
2. 环境变量：`export TF_VAR_vpc_eip_publicip_type=5_bgp`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 7. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建Anti-DDoS基础防护
4. 运行 `terraform show` 查看已创建的Anti-DDoS基础防护详情

## 参考信息

- [华为云Anti-DDoS产品文档](https://support.huaweicloud.com/antiddos/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Anti-DDoS基础防护最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/antiddos/basic)
