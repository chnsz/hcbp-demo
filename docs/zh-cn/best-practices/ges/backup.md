# 部署图备份

## 应用场景

图引擎服务（Graph Engine Service，GES）是华为云提供的一站式图数据管理与分析服务，支持千亿级点边的图数据存储与毫秒级查询分析，广泛应用于社交网络、知识图谱、金融风控、推荐系统等场景。图数据承载了业务的关键关系信息，一旦发生误操作或数据损坏，将直接影响业务分析结果的准确性。

本最佳实践将介绍如何使用Terraform自动化部署GES图实例并创建图备份，包括VPC、子网、安全组的创建，图实例的配置以及图备份的创建。通过为图实例创建备份，可以在需要时将图数据恢复到备份点，保障图数据的安全性与业务的连续性。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [图实例（huaweicloud_ges_graph）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ges_graph)
- [图备份（huaweicloud_ges_backup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/ges_backup)

### 资源/数据源依赖关系

```
huaweicloud_vpc
    └── huaweicloud_vpc_subnet
            └── huaweicloud_ges_graph
                    └── huaweicloud_ges_backup

huaweicloud_networking_secgroup
    └── huaweicloud_ges_graph
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建虚拟私有云

在TF文件（如main.tf）中添加以下脚本以创建虚拟私有云：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云资源
variable "vpc_name" {
  description = "The VPC name for the GES graph"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：通过引用输入变量 vpc_name 进行赋值
- **cidr**：通过引用输入变量 vpc_cidr 进行赋值

### 3. 创建虚拟私有云子网

在TF文件（如main.tf）中添加以下脚本以创建虚拟私有云子网：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云子网资源
variable "subnet_name" {
  description = "The subnet name for the GES graph"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
}

variable "gateway_ip" {
  description = "The gateway IP address of the subnet"
  type        = string
}

resource "huaweicloud_vpc_subnet" "test" {
  name       = var.subnet_name
  vpc_id     = huaweicloud_vpc.test.id
  cidr       = var.subnet_cidr
  gateway_ip = var.gateway_ip
}
```

**参数说明**：
- **name**：通过引用输入变量 subnet_name 进行赋值
- **vpc_id**：通过引用虚拟私有云资源的ID进行赋值
- **cidr**：通过引用输入变量 subnet_cidr 进行赋值
- **gateway_ip**：通过引用输入变量 gateway_ip 进行赋值

### 4. 创建安全组

在TF文件（如main.tf）中添加以下脚本以创建安全组：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建安全组资源
variable "security_group_name" {
  description = "The security group name for the GES graph"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name = var.security_group_name
}
```

**参数说明**：
- **name**：通过引用输入变量 security_group_name 进行赋值

### 5. 创建图实例

在TF文件（如main.tf）中添加以下脚本以创建图实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建图实例资源
variable "graph_name" {
  description = "The GES graph name"
  type        = string
}

variable "graph_size_type_index" {
  description = "The graph size type index"
  type        = string
  default     = "1"
}

variable "graph_cpu_arch" {
  description = "The CPU architecture type of the GES graph"
  type        = string
  default     = "x86_64"
}

variable "graph_crypt_algorithm" {
  description = "The cryptography algorithm of the GES graph"
  type        = string
}

variable "graph_enable_https" {
  description = "Whether to enable HTTPS for the GES graph"
  type        = bool
  default     = false
}

variable "graph_tags" {
  description = "The key/value pairs to associate with the GES graph"
  type        = map(string)
  default     = {
    key = "val"
    foo = "bar"
  }
  nullable    = false
}

resource "huaweicloud_ges_graph" "test" {
  name                  = var.graph_name
  graph_size_type_index = var.graph_size_type_index
  cpu_arch              = var.graph_cpu_arch
  vpc_id                = huaweicloud_vpc.test.id
  subnet_id             = huaweicloud_vpc_subnet.test.id
  security_group_id     = huaweicloud_networking_secgroup.test.id
  crypt_algorithm       = var.graph_crypt_algorithm
  enable_https          = var.graph_enable_https

  tags = var.graph_tags
}
```

**参数说明**：
- **name**：通过引用输入变量 graph_name 进行赋值
- **graph_size_type_index**：通过引用输入变量 graph_size_type_index 进行赋值
- **cpu_arch**：通过引用输入变量 graph_cpu_arch 进行赋值
- **vpc_id**：通过引用虚拟私有云资源的ID进行赋值
- **subnet_id**：通过引用虚拟私有云子网资源的ID进行赋值
- **security_group_id**：通过引用安全组资源的ID进行赋值
- **crypt_algorithm**：通过引用输入变量 graph_crypt_algorithm 进行赋值
- **enable_https**：通过引用输入变量 graph_enable_https 进行赋值
- **tags**：通过引用输入变量 graph_tags 进行赋值

### 6. 创建图备份

在TF文件（如main.tf）中添加以下脚本以创建图备份：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建图备份资源
resource "huaweicloud_ges_backup" "test" {
  graph_id = huaweicloud_ges_graph.test.id
}
```

**参数说明**：
- **graph_id**：通过引用图实例资源的ID进行赋值

### 7. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 资源变量
vpc_name              = "tf_test_ges_vpc"
vpc_cidr              = "192.168.0.0/16"
subnet_name           = "tf_test_ges_subnet"
subnet_cidr           = "192.168.0.0/24"
gateway_ip            = "192.168.0.1"
security_group_name   = "tf_test_ges_secgroup"
graph_name            = "tf_test_ges_graph"
graph_size_type_index = "1"
graph_cpu_arch        = "x86_64"
graph_crypt_algorithm = "generalCipher"
graph_enable_https    = false
graph_tags = {
  key = "val"
  foo = "bar"
}
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

### 8. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建图备份
4. 运行 `terraform show` 查看已创建的图备份

> 注意：图实例的创建大约需要30分钟，备份将在图实例就绪后创建；图实例删除时，其关联的备份也会被自动删除。

## 参考信息

- [华为云图引擎服务产品文档](https://support.huaweicloud.com/ges/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [GES图备份最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/ges/backup)
