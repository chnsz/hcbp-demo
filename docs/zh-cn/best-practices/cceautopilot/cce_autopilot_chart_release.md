# 部署CCE Autopilot应用

## 应用场景

云容器引擎（Cloud Container Engine，CCE）Autopilot是华为云提供的Serverless化Kubernetes集群形态，将控制面和节点资源完全托管，用户无需创建和管理节点即可运行容器化应用。在实际业务中，用户通常需要将打包好的Helm Chart上传至集群，并以Release的形式将应用部署到指定命名空间，从而完成云原生应用的发布与升级。

本最佳实践将介绍如何使用Terraform自动化完成CCE Autopilot集群的创建、Helm Chart的上传以及Release的部署，包括VPC和子网创建、CCE Autopilot集群创建、Chart上传和Release部署。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [CCE Autopilot集群（huaweicloud_cce_autopilot_cluster）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_autopilot_cluster)
- [CCE Autopilot Chart（huaweicloud_cce_autopilot_chart）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_autopilot_chart)
- [CCE Autopilot Release（huaweicloud_cce_autopilot_release）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_autopilot_release)

### 资源/数据源依赖关系

```
huaweicloud_vpc
    └── huaweicloud_vpc_subnet
            └── huaweicloud_cce_autopilot_cluster
                    └── huaweicloud_cce_autopilot_release
                            └── huaweicloud_cce_autopilot_chart
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建VPC

在TF文件（如main.tf）中添加以下脚本以创建VPC：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC
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
- **name**：VPC名称，通过引用输入变量 vpc_name 进行赋值
- **cidr**：VPC的网段，通过引用输入变量 vpc_cidr 进行赋值

### 3. 创建子网

在TF文件（如main.tf）中添加以下脚本以创建子网：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建子网
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
- **vpc_id**：子网所属的VPC ID，引用前一步创建的VPC的ID进行赋值
- **name**：子网名称，通过引用输入变量 subnet_name 进行赋值
- **cidr**：子网的网段，通过引用输入变量 subnet_cidr 进行赋值；当该变量为空时，基于VPC网段自动划分子网网段
- **gateway_ip**：子网的网关IP，通过引用输入变量 gateway_ip 进行赋值；当该变量为空时，基于子网网段自动计算网关IP

### 4. 创建CCE Autopilot集群

在TF文件（如main.tf）中添加以下脚本以创建CCE Autopilot集群：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CCE Autopilot集群
variable "cluster_name" {
  description = "The CCE autopilot cluster name"
  type        = string
}

variable "cluster_version" {
  description = "The Kubernetes version of the cluster"
  type        = string
  default     = "v1.34"
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
  version     = var.cluster_version
  alias       = var.cluster_alias
  description = var.cluster_description
  category    = var.cluster_category
  type        = var.cluster_type
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
}
```

**参数说明**：
- **name**：集群名称，通过引用输入变量 cluster_name 进行赋值
- **flavor**：集群规格，固定为 cce.autopilot.cluster
- **version**：集群的Kubernetes版本，通过引用输入变量 cluster_version 进行赋值
- **alias**：集群别名，通过引用输入变量 cluster_alias 进行赋值
- **description**：集群描述，通过引用输入变量 cluster_description 进行赋值
- **category**：集群类型，通过引用输入变量 cluster_category 进行赋值
- **type**：集群控制节点架构，通过引用输入变量 cluster_type 进行赋值
- **custom_san**：集群API Server证书的自定义SAN字段，通过引用输入变量 cluster_custom_san 进行赋值
- **eip_id**：集群的EIP ID，通过引用输入变量 cluster_eip_id 进行赋值
- **enable_snat**：是否为集群配置SNAT，通过引用输入变量 cluster_enable_snat 进行赋值
- **enable_swr_image_access**：是否启用SWR镜像访问，通过引用输入变量 cluster_enable_swr_image_access 进行赋值
- **delete_efs**：删除集群时是否删除关联的EFS，通过引用输入变量 cluster_delete_efs 进行赋值
- **delete_eni**：删除集群时是否删除关联的ENI，通过引用输入变量 cluster_delete_eni 进行赋值
- **delete_net**：删除集群时是否删除关联的网络资源，通过引用输入变量 cluster_delete_net 进行赋值
- **delete_obs**：删除集群时是否删除关联的OBS，通过引用输入变量 cluster_delete_obs 进行赋值
- **delete_sfs30**：删除集群时是否删除关联的SFS Turbo，通过引用输入变量 cluster_delete_sfs_turbo 进行赋值
- **lts_reclaim_policy**：LTS回收策略，通过引用输入变量 cluster_lts_reclaim_policy 进行赋值
- **host_network**：集群的主机网络配置，vpc 和 subnet 分别引用前一步创建的VPC和子网的ID进行赋值
- **container_network**：集群的容器网络配置，mode 固定为 eni
- **eni_network**：集群的ENI网络配置，subnet_id 引用前一步创建的子网的IPv4子网ID进行赋值
- **extend_param**：集群的扩展参数，enterprise_project_id 通过引用输入变量 cluster_enterprise_project_id 进行赋值
- **configurations_override**：集群组件配置覆盖项，通过引用输入变量 cluster_configurations_override 和 cluster_configurations_override_name 进行赋值
- **tags**：集群标签，通过引用输入变量 cluster_tags 进行赋值

### 5. 上传Helm Chart

在TF文件（如main.tf）中添加以下脚本以上传Helm Chart：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下上传Helm Chart
variable "chart_content" {
  description = "The path of the chart package to be uploaded"
  type        = string
}

variable "chart_parameters" {
  description = "The parameters of the chart"
  type        = string
  default     = "{\"override\":true,\"skip_lint\":true,\"source\":\"package\"}"
}

resource "huaweicloud_cce_autopilot_chart" "test" {
  content    = var.chart_content
  parameters = var.chart_parameters
}
```

**参数说明**：
- **content**：待上传的Chart包路径，通过引用输入变量 chart_content 进行赋值
- **parameters**：Chart上传参数，通过引用输入变量 chart_parameters 进行赋值

### 6. 部署Helm Release

在TF文件（如main.tf）中添加以下脚本以部署Helm Release：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下部署Helm Release
variable "release_name" {
  description = "The name of the release"
  type        = string
}

variable "release_namespace" {
  description = "The namespace to deploy the release"
  type        = string
  default     = "default"
}

variable "release_version" {
  description = "The version of the release"
  type        = string
}

variable "release_description" {
  description = "The description of the release"
  type        = string
  default     = ""
}

variable "release_action" {
  description = "The release updating action. Valid values are: upgrade, rollback"
  type        = string
  default     = ""
}

variable "release_image_tag" {
  description = "The image tag"
  type        = string
  default     = "latest"
}

variable "release_image_pull_policy" {
  description = "The image pull policy"
  type        = string
  default     = "IfNotPresent"
}

variable "release_dry_run" {
  description = "Whether to dry run"
  type        = bool
  default     = false
}

variable "release_name_template" {
  description = "The release name template"
  type        = string
  default     = ""
}

variable "release_no_hooks" {
  description = "Whether to disable hooks during installation"
  type        = bool
  default     = false
}

variable "release_replace" {
  description = "Whether to replace the release with the same name"
  type        = bool
  default     = false
}

variable "release_recreate" {
  description = "Whether to rebuild the release"
  type        = bool
  default     = false
}

variable "release_reset_values" {
  description = "Whether to reset values during an update"
  type        = bool
  default     = false
}

variable "release_rollback_version" {
  description = "The version of the rollback release"
  type        = number
  default     = 0
}

variable "release_include_hooks" {
  description = "Whether to enable hooks during an update or deletion"
  type        = bool
  default     = false
}

resource "huaweicloud_cce_autopilot_release" "test" {
  cluster_id  = huaweicloud_cce_autopilot_cluster.test.id
  chart_id    = huaweicloud_cce_autopilot_chart.test.id
  name        = var.release_name
  namespace   = var.release_namespace
  version     = var.release_version
  description = var.release_description
  action      = var.release_action

  values {
    image_tag         = var.release_image_tag
    image_pull_policy = var.release_image_pull_policy
  }

  parameters {
    dry_run         = var.release_dry_run
    name_template   = var.release_name_template
    no_hooks        = var.release_no_hooks
    replace         = var.release_replace
    recreate        = var.release_recreate
    reset_values    = var.release_reset_values
    release_version = var.release_rollback_version
    include_hooks   = var.release_include_hooks
  }

  depends_on = [
    huaweicloud_cce_autopilot_cluster.test,
    huaweicloud_cce_autopilot_chart.test
  ]
}
```

**参数说明**：
- **cluster_id**：Release所属的集群ID，引用前一步创建的CCE Autopilot集群的ID进行赋值
- **chart_id**：Release使用的Chart ID，引用前一步上传的Chart的ID进行赋值
- **name**：Release名称，通过引用输入变量 release_name 进行赋值
- **namespace**：Release部署的命名空间，通过引用输入变量 release_namespace 进行赋值
- **version**：Release版本，通过引用输入变量 release_version 进行赋值
- **description**：Release描述，通过引用输入变量 release_description 进行赋值
- **action**：Release更新动作，通过引用输入变量 release_action 进行赋值
- **values**：Release的values配置，image_tag 和 image_pull_policy 分别通过引用输入变量 release_image_tag 和 release_image_pull_policy 进行赋值
- **parameters**：Release的部署参数，各字段分别通过引用输入变量 release_dry_run、release_name_template、release_no_hooks、release_replace、release_recreate、release_reset_values、release_rollback_version 和 release_include_hooks 进行赋值

### 7. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 集群变量
vpc_name        = "tf_test_vpc"
subnet_name     = "tf_test_subnet"
cluster_name    = "tf-test-autopilot-cluster"
chart_content   = "./your-chart-version.tgz"
release_name    = "my-release"
release_version = "1.0.0"
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
3. 确认资源计划无误后，运行 `terraform apply` 开始创建CCE Autopilot集群并部署应用
4. 运行 `terraform show` 查看已创建的CCE Autopilot集群及Release

## 参考信息

- [华为云云容器引擎产品文档](https://support.huaweicloud.com/cce/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CCE Autopilot应用最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cceautopilot/cce-autopilot-chart-release)
