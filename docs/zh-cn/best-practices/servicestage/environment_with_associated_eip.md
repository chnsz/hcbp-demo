# 部署关联弹性公网IP的环境

## 应用场景

应用管理与运维平台（ServiceStage）是华为云提供的一站式应用托管平台，环境（Environment）是承载应用运行的基础设施单元，用于将计算、网络等资源纳入统一的应用部署与运维体系。在实际业务中，环境内的应用往往需要访问公网或被公网访问，此时需要为环境关联弹性公网IP（EIP），以便应用能够通过公网地址对外提供服务。

本最佳实践将介绍如何使用Terraform自动化创建VPC、弹性公网IP与ServiceStage环境，并通过环境关联资源将EIP绑定到该环境，从而为环境提供公网访问能力。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [弹性公网IP（huaweicloud_vpc_eip）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [ServiceStage环境（huaweicloud_servicestagev3_environment）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/servicestagev3_environment)
- [ServiceStage环境关联资源（huaweicloud_servicestagev3_environment_associate）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/servicestagev3_environment_associate)

### 资源/数据源依赖关系

```
huaweicloud_vpc
    └── huaweicloud_servicestagev3_environment
            └── huaweicloud_servicestagev3_environment_associate

huaweicloud_vpc_eip
    └── huaweicloud_servicestagev3_environment_associate
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建虚拟私有云

在TF文件（如main.tf）中添加以下脚本以创建VPC：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云资源
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = "192.168.0.0/16"
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量 vpc_name 进行赋值
- **cidr**：VPC的网段，此处固定为 `192.168.0.0/16`

### 3. 创建弹性公网IP

在TF文件（如main.tf）中添加以下脚本以创建EIP：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建弹性公网IP资源
variable "eip_bandwidth_name" {
  description = "The name of the dedicated bandwidth of the EIP"
  type        = string
}

resource "huaweicloud_vpc_eip" "test" {
  publicip {
    type = "5_bgp"
  }

  bandwidth {
    name        = var.eip_bandwidth_name
    size        = 5
    share_type  = "PER"
    charge_mode = "traffic"
  }
}
```

**参数说明**：
- **publicip.type**：公网IP类型，此处为 `5_bgp`（全动态BGP）
- **bandwidth.name**：独享带宽名称，通过引用输入变量 eip_bandwidth_name 进行赋值
- **bandwidth.size**：带宽大小，单位为Mbit/s，此处为 `5`
- **bandwidth.share_type**：带宽类型，`PER` 表示独享带宽
- **bandwidth.charge_mode**：计费模式，`traffic` 表示按流量计费

### 4. 创建ServiceStage环境

在TF文件（如main.tf）中添加以下脚本以创建ServiceStage环境：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ServiceStage环境资源
variable "environment_name" {
  description = "The name of the environment"
  type        = string
}

variable "environment_description" {
  description = "The description of the environment"
  type        = string
  default     = ""
}

resource "huaweicloud_servicestagev3_environment" "test" {
  name        = var.environment_name
  description = var.environment_description
  vpc_id      = huaweicloud_vpc.test.id
}
```

**参数说明**：
- **name**：环境名称，通过引用输入变量 environment_name 进行赋值，长度需为2到64个字符，仅允许字母、数字、中划线（-）和下划线（_），且必须以字母开头、以字母或数字结尾
- **description**：环境描述，通过引用输入变量 environment_description 进行赋值，长度不超过128个字符
- **vpc_id**：环境所属的VPC ID，引用上一步创建的VPC的ID；该参数不支持更新，变更后会重建环境

### 5. 关联弹性公网IP到环境

在TF文件（如main.tf）中添加以下脚本以将EIP关联到环境：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建ServiceStage环境关联资源
resource "huaweicloud_servicestagev3_environment_associate" "test" {
  environment_id = huaweicloud_servicestagev3_environment.test.id

  resources {
    id   = huaweicloud_vpc_eip.test.id
    type = "eip"
  }
}
```

**参数说明**：
- **environment_id**：需要关联资源的环境ID，引用上一步创建的环境的ID；该参数不支持更新
- **resources.id**：待关联资源的ID，此处引用前面创建的EIP的ID
- **resources.type**：待关联资源的类型，此处为 `eip`

> 注意：该关联资源的创建、更新和删除均使用PUT方法以全量模式管理关联资源，请勿同时在控制台向同一环境关联其他资源，否则可能被Terraform覆盖或解除关联。销毁该关联资源仅会将资源从环境中解除关联，EIP本身仍由其自身的资源定义管理。

### 6. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
vpc_name                = "tf_test_ss_vpc"
eip_bandwidth_name      = "tf_test_ss_eip_bandwidth"
environment_name        = "tf_test_ss_environment"
environment_description = "Created by Terraform for ServiceStage best practice example"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="vpc_name=my-vpc"`
2. 环境变量：`export TF_VAR_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 7. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建关联弹性公网IP的ServiceStage环境
4. 运行 `terraform show` 查看已创建的关联弹性公网IP的ServiceStage环境

## 参考信息

- [华为云ServiceStage产品文档](https://support.huaweicloud.com/servicestage/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [ServiceStage关联弹性公网IP的环境最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/servicestage/environment-with-associated-eip)
