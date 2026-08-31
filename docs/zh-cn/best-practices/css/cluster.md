# 部署集群

## 应用场景

云搜索服务（Cloud Search Service，简称CSS）是华为云基于Elasticsearch、OpenSearch构建的完全托管式在线分布式搜索服务，支持结构化、非结构化文本及AI向量的高效检索与分析。通过创建CSS集群，可快速部署具备网络隔离、安全访问控制和灵活计费方式的搜索与分析能力，适用于日志分析、智能客服、知识库问答等多种业务场景。

本最佳实践将介绍如何使用Terraform自动化部署一个CSS集群，包括创建VPC、子网、安全组，以及配置集群引擎类型、安全模式、HTTPS访问和计费方式等参数。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [可用区列表查询数据源（data.huaweicloud_availability_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/availability_zones)
- [CSS规格列表查询数据源（data.huaweicloud_css_flavors）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/css_flavors)

### 资源

- [VPC资源（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网资源（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [CSS集群资源（huaweicloud_css_cluster）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/css_cluster)

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_css_cluster

data.huaweicloud_css_flavors
    └── huaweicloud_css_cluster

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_css_cluster

huaweicloud_networking_secgroup
    └── huaweicloud_css_cluster
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 通过数据源查询CSS集群创建所需的可用区

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建CSS集群：

```hcl
variable "availability_zone" {
  description = "The availability zone to which the CSS cluster belongs"
  type        = string
  default     = ""
  nullable    = false
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建CSS集群
data "huaweicloud_availability_zones" "test" {
  count = var.availability_zone == "" ? 1 : 0
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行可用区列表查询数据源，仅当 `var.availability_zone` 为空时创建数据源（即执行可用区列表查询）

### 3. 创建VPC资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC资源：

```hcl
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC资源，用于部署CSS集群
resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量vpc_name进行赋值
- **cidr**：VPC的CIDR网段，通过引用输入变量vpc_cidr进行赋值，默认值为"192.168.0.0/16"

### 4. 创建VPC子网资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建VPC子网资源：

```hcl
variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet"
  type        = string
  default     = ""
  nullable    = false
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC子网资源，用于部署CSS集群
resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.subnet_gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.subnet_gateway_ip
}
```

**参数说明**：
- **vpc_id**：子网所属的VPC ID，引用前面创建的VPC资源（huaweicloud_vpc.test）的ID
- **name**：子网名称，通过引用输入变量subnet_name进行赋值
- **cidr**：子网的CIDR网段，当subnet_cidr为空时使用cidrsubnet函数从VPC的CIDR网段中划分，否则通过引用输入变量subnet_cidr进行赋值
- **gateway_ip**：子网的网关IP，当subnet_gateway_ip为空时使用cidrhost函数从子网网段中获取，否则通过引用输入变量subnet_gateway_ip进行赋值

### 5. 创建安全组资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建安全组资源：

```hcl
variable "security_group_name" {
  description = "The name of the security group"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源，用于部署CSS集群
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组名称，通过引用输入变量security_group_name进行赋值
- **delete_default_rules**：是否删除默认规则，设置为true

### 6. 通过数据源查询CSS集群创建所需的规格

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建CSS集群：

```hcl
variable "cluster_flavor" {
  description = "The flavor of the CSS cluster"
  type        = string
  default     = ""
  nullable    = false
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下所有的CSS规格信息，用于创建CSS集群
data "huaweicloud_css_flavors" "test" {
  count = var.cluster_flavor == "" ? 1 : 0
}
```

**参数说明**：
- **count**：数据源的创建数，用于控制是否执行CSS规格列表查询数据源，仅当 `var.cluster_flavor` 为空时创建数据源（即执行CSS规格列表查询）

### 7. 创建CSS集群

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建CSS集群资源：

```hcl
variable "cluster_name" {
  description = "The name of the CSS cluster"
  type        = string
}

variable "cluster_engine_version" {
  description = "The engine version of the CSS cluster"
  type        = string
  default     = "7.10.2"
}

variable "cluster_instance_number" {
  description = "The number of instances of the CSS cluster"
  type        = number
  default     = 3
}

variable "cluster_volume_type" {
  description = "The volume type of the CSS cluster"
  type        = string
  default     = "ULTRAHIGH"
}

variable "cluster_volume_size" {
  description = "The volume size of the CSS cluster"
  type        = number
  default     = 40
}

variable "cluster_engine_type" {
  description = "The engine type of the CSS cluster"
  type        = string
  default     = "elasticsearch"
}

variable "cluster_security_mode" {
  description = "Whether to enable the security mode of the CSS cluster"
  type        = bool
  default     = false
}

variable "cluster_access_password" {
  description = "The access password of the CSS cluster"
  sensitive   = true
  type        = string
  default     = ""
}

variable "cluster_https_enabled" {
  description = "Whether to enable HTTPS access of the CSS cluster"
  type        = bool
  default     = false
}

variable "charging_mode" {
  description = "The charging mode of the CSS cluster"
  type        = string
  default     = "postPaid"
}

variable "period_unit" {
  description = "The period unit of the CSS cluster"
  type        = string
  default     = null
}

variable "period" {
  description = "The period of the CSS cluster"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "The auto renew of the CSS cluster"
  type        = string
  default     = "false"
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CSS集群资源
resource "huaweicloud_css_cluster" "test" {
  name              = var.cluster_name
  engine_version    = var.cluster_engine_version
  availability_zone = var.availability_zone == "" ? try(data.huaweicloud_availability_zones.test[0].names[0], null) : var.availability_zone
  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id

  ess_node_config {
    flavor          = var.cluster_flavor == "" ? try(data.huaweicloud_css_flavors.test[0].flavors[0].name, null) : var.cluster_flavor
    instance_number = var.cluster_instance_number

    volume {
      volume_type = var.cluster_volume_type
      size        = var.cluster_volume_size
    }
  }

  engine_type   = var.cluster_engine_type
  security_mode = var.cluster_security_mode
  password      = var.cluster_access_password
  https_enabled = var.cluster_https_enabled
  charging_mode = var.charging_mode
  period_unit   = var.period_unit
  period        = var.period
  auto_renew    = var.auto_renew
}
```

**参数说明**：
- **name**：CSS集群名称，通过引用输入变量cluster_name进行赋值
- **engine_version**：CSS集群引擎版本，通过引用输入变量cluster_engine_version进行赋值，默认为"7.10.2"
- **availability_zone**：CSS集群所在可用区，当availability_zone为空时根据可用区列表查询数据源（data.huaweicloud_availability_zones）的返回结果进行赋值，否则通过引用输入变量availability_zone进行赋值
- **vpc_id**：CSS集群所属VPC的ID，引用前面创建的VPC资源（huaweicloud_vpc.test）的ID
- **subnet_id**：CSS集群所属子网的ID，引用前面创建的VPC子网资源（huaweicloud_vpc_subnet.test）的ID
- **security_group_id**：CSS集群关联的安全组ID，引用前面创建的安全组资源（huaweicloud_networking_secgroup.test）的ID
- **ess_node_config**：CSS集群节点配置块
  - **flavor**：CSS集群节点规格，当cluster_flavor为空时根据CSS规格列表查询数据源（data.huaweicloud_css_flavors）的返回结果进行赋值，否则通过引用输入变量cluster_flavor进行赋值
  - **instance_number**：CSS集群节点数量，通过引用输入变量cluster_instance_number进行赋值，默认为3
  - **volume**：节点存储配置块
    - **volume_type**：存储类型，通过引用输入变量cluster_volume_type进行赋值，默认为"ULTRAHIGH"
    - **size**：存储容量，通过引用输入变量cluster_volume_size进行赋值，默认为40
- **engine_type**：CSS集群引擎类型，通过引用输入变量cluster_engine_type进行赋值，默认为"elasticsearch"
- **security_mode**：是否开启安全模式，通过引用输入变量cluster_security_mode进行赋值，默认为false
- **password**：CSS集群访问密码，通过引用输入变量cluster_access_password进行赋值，默认为空字符串
- **https_enabled**：是否开启HTTPS访问，通过引用输入变量cluster_https_enabled进行赋值，默认为false
- **charging_mode**：计费模式，通过引用输入变量charging_mode进行赋值，默认为"postPaid"
- **period_unit**：订购周期单位，通过引用输入变量period_unit进行赋值，默认为null
- **period**：订购周期，通过引用输入变量period进行赋值，默认为null
- **auto_renew**：是否自动续费，通过引用输入变量auto_renew进行赋值，默认为"false"

### 8. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 网络配置
vpc_name            = "tf_test_cluster"
subnet_name         = "tf_test_cluster"
security_group_name = "tf_test_cluster"

# CSS集群配置
cluster_name = "tf_test_cluster"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="cluster_name=tf_test_cluster" -var="vpc_name=tf_test_cluster"`
2. 环境变量：`export TF_VAR_cluster_name=tf_test_cluster`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 9. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建CSS集群
4. 运行 `terraform show` 查看已创建的CSS集群

## 参考信息

- [华为云CSS产品文档](https://support.huaweicloud.com/css/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CSS集群最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/css/css-cluster)
