# 部署共享带宽上的弹性公网IP

## 应用场景

弹性公网IP（Elastic IP，EIP）是华为云提供的一种可独立申请和绑定的公网IP地址资源，为云上资源提供访问公网和被公网访问的能力。共享带宽可以将多个EIP加入同一份带宽资源中，实现带宽的复用与共享，从而降低公网带宽成本并提升带宽利用率。

本最佳实践将介绍如何使用Terraform创建共享带宽，并在创建弹性公网IP时直接将其挂载到该共享带宽上，使EIP在创建阶段即使用共享带宽的公网出口能力，适用于需要为多个公网IP统一规划与复用带宽的组网场景。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [共享带宽（huaweicloud_vpc_bandwidth）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_bandwidth)
- [弹性公网IP（huaweicloud_vpc_eip）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)

### 资源/数据源依赖关系

```
huaweicloud_vpc_bandwidth
    └── huaweicloud_vpc_eip
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建共享带宽

在TF文件（如main.tf）中添加以下脚本以创建共享带宽：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建共享带宽资源
variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = ""
  nullable    = false
}

variable "bandwidth_name" {
  description = "The name of the shared bandwidth"
  type        = string
}

variable "bandwidth_size" {
  description = "The size of the shared bandwidth in Mbit/s"
  type        = number
  default     = 5
}

variable "bandwidth_charge_mode" {
  description = "The charge mode of the shared bandwidth"
  type        = string
  default     = "bandwidth"
}

variable "bandwidth_type" {
  description = "The type of the bandwidth"
  type        = string
  default     = "share"
}

variable "bandwidth_public_border_group" {
  description = "The border group of the public IP"
  type        = string
  default     = "center"
}

resource "huaweicloud_vpc_bandwidth" "test" {
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
  name                  = var.bandwidth_name
  size                  = var.bandwidth_size
  charge_mode           = var.bandwidth_charge_mode
  bandwidth_type        = var.bandwidth_type
  public_border_group   = var.bandwidth_public_border_group
}
```

**参数说明**：
- **enterprise_project_id**：通过引用输入变量 enterprise_project_id 进行赋值，当取值为空字符串时使用null表示不指定企业项目
- **name**：通过引用输入变量 bandwidth_name 进行赋值，用于指定共享带宽的名称
- **size**：通过引用输入变量 bandwidth_size 进行赋值，用于指定共享带宽的大小，单位为Mbit/s
- **charge_mode**：通过引用输入变量 bandwidth_charge_mode 进行赋值，用于指定共享带宽的计费模式
- **bandwidth_type**：通过引用输入变量 bandwidth_type 进行赋值，用于指定共享带宽的类型
- **public_border_group**：通过引用输入变量 bandwidth_public_border_group 进行赋值，用于指定公网IP的边界组

### 3. 创建挂载到共享带宽的弹性公网IP

在TF文件（如main.tf）中添加以下脚本以创建弹性公网IP，并在其`bandwidth`块中通过`share_type = "WHOLE"`与共享带宽ID将其直接挂载到上一步创建的共享带宽：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建弹性公网IP资源
variable "eip_type" {
  description = "The type of the EIP"
  type        = string
  default     = "5_bgp"
}

variable "eip_description" {
  description = "The description of the EIP"
  type        = string
  default     = ""
}

variable "eip_tags" {
  description = "The tags of the EIP"
  type        = map(string)
  default     = null
}

resource "huaweicloud_vpc_eip" "test" {
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null

  publicip {
    type = var.eip_type
  }

  bandwidth {
    share_type = "WHOLE"
    id         = huaweicloud_vpc_bandwidth.test.id
  }

  description = var.eip_description
  tags        = var.eip_tags
}
```

**参数说明**：
- **enterprise_project_id**：通过引用输入变量 enterprise_project_id 进行赋值，当取值为空字符串时使用null表示不指定企业项目
- **publicip.type**：通过引用输入变量 eip_type 进行赋值，用于指定弹性公网IP的类型
- **bandwidth.share_type**：设置为`WHOLE`，表示该弹性公网IP使用共享带宽
- **bandwidth.id**：通过引用共享带宽资源`huaweicloud_vpc_bandwidth.test`的ID进行赋值，用于将弹性公网IP挂载到指定的共享带宽
- **description**：通过引用输入变量 eip_description 进行赋值，用于指定弹性公网IP的描述信息
- **tags**：通过引用输入变量 eip_tags 进行赋值，用于指定弹性公网IP的标签

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 共享带宽名称
bandwidth_name = "tf_test_shared_bandwidth"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="bandwidth_name=my-bandwidth"`
2. 环境变量：`export TF_VAR_bandwidth_name=my-bandwidth`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建共享带宽及挂载到共享带宽的弹性公网IP
4. 运行 `terraform show` 查看已创建的共享带宽及挂载到共享带宽的弹性公网IP

## 参考信息

- [华为云弹性公网IP产品文档](https://support.huaweicloud.com/eip/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [弹性公网IP共享带宽最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/eip/eip-with-shared-bandwidth)
