# 部署CCE Autopilot集群

## 应用场景

云容器引擎（Cloud Container Engine，CCE）Autopilot是华为云提供的Serverless化Kubernetes集群形态，将控制面和节点资源完全托管，用户无需创建和管理节点，只需提交工作负载即可运行容器化应用。Autopilot集群采用ENI容器网络模式，支持Kubernetes社区原生API与工具，适用于希望降低集群运维成本、按实际使用量付费的云原生业务场景。

本最佳实践将介绍如何使用Terraform自动化部署一个CCE Autopilot集群，包括VPC和子网的创建以及CCE Autopilot集群的创建，帮助您快速上手CCE Autopilot集群的自动化部署。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [CCE Autopilot集群（huaweicloud_cce_autopilot_cluster）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_autopilot_cluster)

### 资源/数据源依赖关系

```
huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_cce_autopilot_cluster
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
  description = "The VPC name"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：通过引用输入变量 vpc_name 进行赋值
- **cidr**：通过引用输入变量 vpc_cidr 进行赋值，默认值为 192.168.0.0/16

### 3. 创建虚拟私有云子网

在TF文件（如main.tf）中添加以下脚本以创建虚拟私有云子网：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建虚拟私有云子网资源
variable "subnet_name" {
  description = "The subnet name"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
}

variable "gateway_ip" {
  description = "The gateway IP address of the subnet"
  type        = string
  default     = ""
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.gateway_ip
}
```

**参数说明**：
- **vpc_id**：通过引用虚拟私有云资源的ID进行赋值
- **name**：通过引用输入变量 subnet_name 进行赋值
- **cidr**：通过引用输入变量 subnet_cidr 进行赋值，当取值为空时基于VPC的CIDR自动划分子网网段
- **gateway_ip**：通过引用输入变量 gateway_ip 进行赋值，当取值为空时基于子网网段自动计算网关地址

### 4. 创建CCE Autopilot集群

在TF文件（如main.tf）中添加以下脚本以创建CCE Autopilot集群：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CCE Autopilot集群资源
variable "cluster_name" {
  description = "The CCE autopilot cluster name"
  type        = string
}

variable "cluster_alias" {
  description = "The alias of the cluster"
  type        = string
  default     = ""
}

variable "cluster_description" {
  description = "The description of the cluster"
  type        = string
  default     = ""
}

variable "cluster_category" {
  description = "The cluster type. Only Turbo is supported"
  type        = string
  default     = "Turbo"
}

variable "cluster_type" {
  description = "The master node architecture. Valid values are: VirtualMachine"
  type        = string
  default     = "VirtualMachine"
}

variable "cluster_annotations" {
  description = "The cluster annotations in the format of key-value pairs"
  type        = map(string)
  default     = {}
}

variable "cluster_custom_san" {
  description = "The custom SAN field in the API server certificate of the cluster"
  type        = list(string)
  default     = []
}

variable "cluster_eip_id" {
  description = "The EIP ID of the cluster"
  type        = string
  default     = ""
}

variable "cluster_enable_snat" {
  description = "Whether SNAT is configured for the cluster"
  type        = bool
  default     = false
}

variable "cluster_enable_swr_image_access" {
  description = "Whether SWR image access is enabled for the cluster"
  type        = bool
  default     = false
}

variable "cluster_delete_efs" {
  description = "Whether to delete the associated EFS when deleting the cluster"
  type        = bool
  default     = false
}

variable "cluster_delete_eni" {
  description = "Whether to delete the associated ENI when deleting the cluster"
  type        = bool
  default     = false
}

variable "cluster_delete_net" {
  description = "Whether to delete the associated network resources when deleting the cluster"
  type        = bool
  default     = false
}

variable "cluster_delete_obs" {
  description = "Whether to delete the associated OBS when deleting the cluster"
  type        = bool
  default     = false
}

variable "cluster_delete_sfs_turbo" {
  description = "Whether to delete the associated SFS Turbo when deleting the cluster"
  type        = bool
  default     = false
}

variable "cluster_lts_reclaim_policy" {
  description = "The LTS reclaim policy. Valid values are: delete, retain"
  type        = string
  default     = "retain"
}

variable "cluster_service_network_cidr" {
  description = "The IPv4 CIDR of the service network"
  type        = string
  default     = ""
}

variable "cluster_enterprise_project_id" {
  description = "The ID of the enterprise project to which the cluster belongs"
  type        = string
  default     = "0"
}

variable "cluster_configurations_override" {
  description = "The component configuration items for override"
  type        = list(object({
    name  = string
    value = string
  }))
  default     = []
}

variable "cluster_configurations_override_name" {
  description = "The component name for configurations override"
  type        = string
  default     = ""
}

variable "cluster_tags" {
  description = "The tags of the cluster"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_cce_autopilot_cluster" "test" {
  name        = var.cluster_name
  flavor      = "cce.autopilot.cluster"
  alias       = var.cluster_alias
  description = var.cluster_description
  category    = var.cluster_category
  type        = var.cluster_type
  annotations = var.cluster_annotations
  custom_san  = var.cluster_custom_san
  eip_id      = var.cluster_eip_id

  enable_snat             = var.cluster_enable_snat
  enable_swr_image_access = var.cluster_enable_swr_image_access

  delete_efs   = var.cluster_delete_efs
  delete_eni   = var.cluster_delete_eni
  delete_net   = var.cluster_delete_net
  delete_obs   = var.cluster_delete_obs
  delete_sfs30 = var.cluster_delete_sfs_turbo

  lts_reclaim_policy = var.cluster_lts_reclaim_policy

  host_network {
    vpc    = huaweicloud_vpc.test.id
    subnet = huaweicloud_vpc_subnet.test.id
  }

  container_network {
    mode = "eni"
  }

  eni_network {
    subnets {
      subnet_id = huaweicloud_vpc_subnet.test.ipv4_subnet_id
    }
  }

  service_network {
    ipv4_cidr = var.cluster_service_network_cidr
  }

  extend_param {
    enterprise_project_id = var.cluster_enterprise_project_id
  }

  dynamic "configurations_override" {
    for_each = length(var.cluster_configurations_override) > 0 ? [1] : []

    content {
      name = var.cluster_configurations_override_name

      dynamic "configurations" {
        for_each = var.cluster_configurations_override

        content {
          name  = configurations.value.name
          value = configurations.value.value
        }
      }
    }
  }

  tags = var.cluster_tags

  lifecycle {
    ignore_changes = [
      annotations,
      tags["CCE-Cluster-ID"],
      service_network,
    ]
  }
}
```

**参数说明**：
- **name**：通过引用输入变量 cluster_name 进行赋值
- **flavor**：集群规格，固定为 cce.autopilot.cluster
- **alias**：通过引用输入变量 cluster_alias 进行赋值
- **description**：通过引用输入变量 cluster_description 进行赋值
- **category**：通过引用输入变量 cluster_category 进行赋值，默认值为 Turbo
- **type**：通过引用输入变量 cluster_type 进行赋值，默认值为 VirtualMachine
- **annotations**：通过引用输入变量 cluster_annotations 进行赋值
- **custom_san**：通过引用输入变量 cluster_custom_san 进行赋值
- **eip_id**：通过引用输入变量 cluster_eip_id 进行赋值
- **enable_snat**：通过引用输入变量 cluster_enable_snat 进行赋值
- **enable_swr_image_access**：通过引用输入变量 cluster_enable_swr_image_access 进行赋值
- **delete_efs**：通过引用输入变量 cluster_delete_efs 进行赋值
- **delete_eni**：通过引用输入变量 cluster_delete_eni 进行赋值
- **delete_net**：通过引用输入变量 cluster_delete_net 进行赋值
- **delete_obs**：通过引用输入变量 cluster_delete_obs 进行赋值
- **delete_sfs30**：通过引用输入变量 cluster_delete_sfs_turbo 进行赋值
- **lts_reclaim_policy**：通过引用输入变量 cluster_lts_reclaim_policy 进行赋值
- **host_network**：集群的主机网络配置，vpc 和 subnet 分别引用虚拟私有云和子网资源的ID
- **container_network**：集群的容器网络配置，mode 固定为 eni
- **eni_network**：集群的ENI网络配置，subnet_id 引用子网资源的IPv4子网ID
- **service_network**：集群的服务网络配置，ipv4_cidr 通过引用输入变量 cluster_service_network_cidr 进行赋值
- **extend_param**：集群的扩展参数，enterprise_project_id 通过引用输入变量 cluster_enterprise_project_id 进行赋值
- **configurations_override**：集群的组件配置覆盖项，通过引用输入变量 cluster_configurations_override 和 cluster_configurations_override_name 进行赋值
- **tags**：通过引用输入变量 cluster_tags 进行赋值

### 5. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 集群变量
vpc_name            = "tf_test_vpc"
subnet_name         = "tf_test_subnet"
cluster_name        = "tf-test-autopilot-cluster"
cluster_description = "Created by terraform script"
cluster_tags = {
  owner = "terraform"
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

### 6. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建CCE Autopilot集群
4. 运行 `terraform show` 查看已创建的CCE Autopilot集群

## 参考信息

- [华为云云容器引擎产品文档](https://support.huaweicloud.com/cce/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CCE Autopilot集群最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cceautopilot/cce-autopilot-cluster)
