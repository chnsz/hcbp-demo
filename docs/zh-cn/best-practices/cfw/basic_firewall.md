# 部署基础防火墙

## 应用场景

云防火墙（Cloud Firewall，CFW）是新一代的云原生防火墙，提供云上互联网边界和VPC边界的防护，包括实时入侵检测与防御、全局统一访问控制、全流量分析可视化、日志审计与溯源分析等能力。通过购买CFW防火墙实例，并配置EIP自动防护或手动绑定已有EIP，可快速为云上公网资产开启互联网边界防护。

本最佳实践将介绍如何使用Terraform自动化部署一个基础的CFW防火墙，包括购买防火墙实例、开启EIP自动防护，以及按需手动绑定已有EIP进行防护。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [CFW防火墙资源（huaweicloud_cfw_firewall）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cfw_firewall)
- [CFW EIP自动防护资源（huaweicloud_cfw_eip_auto_protection）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cfw_eip_auto_protection)
- [CFW EIP防护资源（huaweicloud_cfw_eip_protection）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cfw_eip_protection)

### 资源/数据源依赖关系

```
huaweicloud_cfw_firewall
    ├── huaweicloud_cfw_eip_auto_protection
    └── huaweicloud_cfw_eip_protection
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建CFW防火墙

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CFW防火墙资源：

```hcl
variable "firewall_name" {
  description = "The CFW firewall name"
  type        = string
}

variable "firewall_flavor" {
  description = "The flavor version of the firewall"
  type        = string
  default     = "Professional"
}

variable "firewall_charging_mode" {
  description = "The charging mode of the firewall"
  type        = string
  default     = "postPaid"
}

variable "firewall_tags" {
  description = "The key/value pairs to associate with the resources"
  type        = map(string)
  default = {
    key = "value"
    foo = "bar"
  }
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CFW防火墙资源
resource "huaweicloud_cfw_firewall" "test" {
  name = var.firewall_name

  flavor {
    version = var.firewall_flavor
  }

  charging_mode = var.firewall_charging_mode
  tags          = var.firewall_tags
}
```

**参数说明**：
- **name**：CFW防火墙名称，通过引用输入变量firewall_name进行赋值
- **flavor**：防火墙规格配置块
  - **version**：防火墙规格版本，通过引用输入变量firewall_flavor进行赋值，默认为"Professional"
- **charging_mode**：防火墙计费模式，通过引用输入变量firewall_charging_mode进行赋值，默认为"postPaid"
- **tags**：关联到防火墙的标签键值对，通过引用输入变量firewall_tags进行赋值

### 3. 创建EIP自动防护

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CFW EIP自动防护资源：

```hcl
variable "eip_auto_protection_status" {
  description = "Whether to enable auto-protection for EIPs. 1: enable, 0: disable"
  type        = number
  default     = 1
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CFW EIP自动防护资源
resource "huaweicloud_cfw_eip_auto_protection" "test" {
  fw_instance_id = huaweicloud_cfw_firewall.test.id
  object_id      = try(huaweicloud_cfw_firewall.test.protect_objects[0].object_id, "")
  status         = var.eip_auto_protection_status
}
```

**参数说明**：
- **fw_instance_id**：防火墙实例ID，引用前面创建的CFW防火墙资源（huaweicloud_cfw_firewall.test）的ID
- **object_id**：防护对象ID，根据前面创建的CFW防火墙资源（huaweicloud_cfw_firewall.test）返回的protect_objects进行赋值
- **status**：是否开启EIP自动防护，通过引用输入变量eip_auto_protection_status进行赋值，1表示开启，0表示关闭，默认为1

### 4. 创建EIP防护

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CFW EIP防护资源，用于手动绑定已有EIP进行防护：

```hcl
variable "eip_protection_enabled" {
  description = "Whether to enable manual EIP protection for specific existing EIPs"
  type        = bool
  default     = false
}

variable "eip_protection_eip_ids" {
  description = "The list of existing EIPs to protect, each with id and public_ipv4"
  type = list(object({
    id          = string
    public_ipv4 = string
  }))
  default  = []
  nullable = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CFW EIP防护资源
resource "huaweicloud_cfw_eip_protection" "test" {
  count = var.eip_protection_enabled && length(var.eip_protection_eip_ids) > 0 ? 1 : 0

  object_id = try(huaweicloud_cfw_firewall.test.protect_objects[0].object_id, "")

  dynamic "protected_eip" {
    for_each = var.eip_protection_eip_ids

    content {
      id          = protected_eip.value.id
      public_ipv4 = protected_eip.value.public_ipv4
    }
  }
}
```

**参数说明**：
- **count**：EIP防护资源的创建数，仅当`var.eip_protection_enabled`为true且`var.eip_protection_eip_ids`不为空时创建
- **object_id**：防护对象ID，根据前面创建的CFW防火墙资源（huaweicloud_cfw_firewall.test）返回的protect_objects进行赋值
- **protected_eip**：受防护EIP配置，通过动态块 `dynamic "protected_eip"` 根据输入变量eip_protection_eip_ids创建
  - **id**：已有EIP的ID，通过引用输入变量中的id进行赋值
  - **public_ipv4**：已有EIP的公网IPv4地址，通过引用输入变量中的public_ipv4进行赋值

> 注意：当仅使用EIP自动防护时，可将`eip_protection_enabled`保持为false；如需手动绑定指定EIP，请将其设置为true，并在`eip_protection_eip_ids`中填写实际EIP信息。

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# CFW防火墙配置
firewall_name          = "tf_test_cfw_firewall"
firewall_flavor        = "Professional"
firewall_charging_mode = "postPaid"
firewall_tags = {
  environment = "test"
  managed_by  = "terraform"
}

# EIP自动防护配置
eip_auto_protection_status = 1

# 手动EIP防护配置
eip_protection_enabled = false
eip_protection_eip_ids = []
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="firewall_name=tf_test_cfw_firewall" -var="firewall_flavor=Professional"`
2. 环境变量：`export TF_VAR_firewall_name=tf_test_cfw_firewall`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建基础CFW防火墙
4. 运行 `terraform show` 查看已创建的基础CFW防火墙

## 参考信息

- [华为云CFW产品文档](https://support.huaweicloud.com/cfw/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CFW基础防火墙最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cfw/basic-firewall)
