# 部署CCE Autopilot集群插件

## 应用场景

云容器引擎（Cloud Container Engine，CCE）Autopilot集群是一种Serverless化的Kubernetes集群形态，用户无需管理节点即可直接运行容器化应用。在实际业务中，集群创建完成后通常还需要安装日志采集、监控等插件，才能满足可观测性与运维需求。

本最佳实践将介绍如何使用Terraform自动化部署一个CCE Autopilot集群，并在该集群上安装log-agent插件。相关操作包括VPC与子网创建、CCE Autopilot集群创建、SWR组织创建以及插件安装。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [CCE Autopilot集群（huaweicloud_cce_autopilot_cluster）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_autopilot_cluster)
- [容器镜像服务组织（huaweicloud_swr_organization）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/swr_organization)
- [CCE Autopilot插件（huaweicloud_cce_autopilot_addon）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_autopilot_addon)

### 资源/数据源依赖关系

```
huaweicloud_vpc
    └── huaweicloud_vpc_subnet
            └── huaweicloud_cce_autopilot_cluster
                    └── huaweicloud_cce_autopilot_addon

huaweicloud_swr_organization
    └── huaweicloud_cce_autopilot_addon
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建虚拟私有云

在TF文件（如main.tf）中添加以下脚本：

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
- **name**：VPC名称，通过引用输入变量 vpc_name 进行赋值
- **cidr**：VPC的网段，通过引用输入变量 vpc_cidr 进行赋值

### 3. 创建虚拟私有云子网

在TF文件（如main.tf）中添加以下脚本：

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
- **vpc_id**：子网所属的VPC ID，引用前一步创建的VPC资源的ID进行赋值
- **name**：子网名称，通过引用输入变量 subnet_name 进行赋值
- **cidr**：子网的网段，通过引用输入变量 subnet_cidr 进行赋值；当该变量为空时，基于VPC网段自动划分子网网段
- **gateway_ip**：子网的网关IP，通过引用输入变量 gateway_ip 进行赋值；当该变量为空时，基于子网网段自动计算网关IP

### 4. 创建CCE Autopilot集群

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CCE Autopilot集群资源
variable "cluster_name" {
  description = "The CCE autopilot cluster name"
  type        = string
}

variable "cluster_version" {
  description = "The Kubernetes version of the cluster"
  type        = string
  default     = "v1.36"
}

variable "cluster_alias" {
  description = "The alias of the cluster"
  type        = string
  default     = ""
}

variable "cluster_description" {
  description = "The description of the cluster"
  type        = string
  default     = "CCE Autopilot cluster for addon deployment"
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
  default     = true
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

  tags = var.cluster_tags

  lifecycle {
    ignore_changes = [
      enable_snat,
      delete_efs,
      delete_obs,
    ]
  }
}
```

**参数说明**：
- **name**：集群名称，通过引用输入变量 cluster_name 进行赋值
- **flavor**：集群规格，固定为 cce.autopilot.cluster
- **version**：集群的Kubernetes版本，通过引用输入变量 cluster_version 进行赋值
- **alias**：集群别名，通过引用输入变量 cluster_alias 进行赋值
- **description**：集群描述，通过引用输入变量 cluster_description 进行赋值
- **category**：集群类型，通过引用输入变量 cluster_category 进行赋值
- **type**：集群Master节点架构，通过引用输入变量 cluster_type 进行赋值
- **custom_san**：API Server证书的自定义SAN字段，通过引用输入变量 cluster_custom_san 进行赋值
- **eip_id**：集群绑定的EIP ID，通过引用输入变量 cluster_eip_id 进行赋值
- **enable_snat**：是否配置SNAT，通过引用输入变量 cluster_enable_snat 进行赋值
- **enable_swr_image_access**：是否启用SWR镜像访问，通过引用输入变量 cluster_enable_swr_image_access 进行赋值
- **delete_efs**、**delete_eni**、**delete_net**、**delete_obs**、**delete_sfs30**：删除集群时是否同时删除关联的EFS、ENI、网络资源、OBS、SFS Turbo，分别通过引用输入变量 cluster_delete_efs、cluster_delete_eni、cluster_delete_net、cluster_delete_obs、cluster_delete_sfs_turbo 进行赋值
- **lts_reclaim_policy**：LTS回收策略，通过引用输入变量 cluster_lts_reclaim_policy 进行赋值
- **host_network**：集群主机网络配置，引用前一步创建的子网资源ID进行赋值
- **container_network**：容器网络配置，本实践中使用eni模式
- **eni_network**：ENI网络配置，引用前一步创建的子网的IPv4子网ID进行赋值
- **extend_param**：扩展参数，其中企业项目ID通过引用输入变量 cluster_enterprise_project_id 进行赋值
- **tags**：集群标签，通过引用输入变量 cluster_tags 进行赋值

### 5. 创建容器镜像服务组织

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建容器镜像服务组织资源
variable "swr_organization_name" {
  description = "The name of the SWR organization"
  type        = string
}

resource "huaweicloud_swr_organization" "test" {
  name = var.swr_organization_name
}
```

**参数说明**：
- **name**：SWR组织名称，通过引用输入变量 swr_organization_name 进行赋值，该名称在华为云全局唯一

### 6. 创建CCE Autopilot插件

在TF文件（如main.tf）中添加以下脚本：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建CCE Autopilot插件资源
variable "addon_template_name" {
  description = "The name of the add-on template to be installed"
  type        = string
  default     = "log-agent"
}

variable "addon_version" {
  description = "The version of the add-on. If not specified, the latest compatible version will be used"
  type        = string
  default     = ""
}

variable "addon_name" {
  description = "The name of the add-on"
  type        = string
  default     = ""
}

variable "addon_alias" {
  description = "The alias of the add-on"
  type        = string
  default     = ""
}

variable "addon_values_basic" {
  description = "The basic configuration values for the add-on"
  type        = map(any)
  default     = {}
}

variable "addon_values_flavor" {
  description = "The flavor configuration values for the add-on"
  type        = map(any)
  default     = {}
}

variable "addon_values_custom" {
  description = "The custom configuration values for the add-on"
  type        = map(any)
  default     = {}
}

resource "huaweicloud_cce_autopilot_addon" "test" {
  cluster_id          = huaweicloud_cce_autopilot_cluster.test.id
  addon_template_name = var.addon_template_name
  version             = var.addon_version
  name                = var.addon_name
  alias               = var.addon_alias

  values = {
    "basic" = jsonencode(merge({
      "aomEndpoint" : "https://aom.${var.region_name}.myhuaweicloud.com",
      "iam_url" : "iam.${var.region_name}.myhuaweicloud.com",
      "ltsAccessEndpoint" : "https://lts-access.${var.region_name}.myhuaweicloud.com:8102",
      "ltsEndpoint" : "https://lts.${var.region_name}.myhuaweicloud.com",
      "region" : var.region_name,
      "swr_addr" : "swr.${var.region_name}.myhuaweicloud.com",
      "swr_user" : huaweicloud_swr_organization.test.name,
      "rbac_enabled" : true,
      "cluster_version" : huaweicloud_cce_autopilot_cluster.test.version
    }, var.addon_values_basic))

    "flavor" = jsonencode(merge({
      "category" : ["Autopilot"],
      "description" : "Applicable to clusters where the logs of a single pod are less than 10000/s or 5 MB/s.",
      "is_default" : true,
      "name" : "custom-resources",
      "resources" : [
        {
          "name" : "log-operator",
          "limitsCpu" : "1000m",
          "requestsCpu" : "1000m",
          "replicas" : 3,
          "limitsMem" : "2048Mi",
          "requestsMem" : "2048Mi"
        },
        {
          "name" : "otel-collector-event",
          "limitsCpu" : "1000m",
          "requestsCpu" : "1000m",
          "replicas" : 3,
          "limitsMem" : "2048Mi",
          "requestsMem" : "2048Mi"
        }
      ],
      "size" : "custom"
    }, var.addon_values_flavor))

    "custom" = jsonencode(merge({
      "accessKey" : "",
      "agency_name" : "",
      "aomEndpoint" : "https://aom.${var.region_name}.myhuaweicloud.com",
      "aomPrivateEndpointIP" : "",
      "bufferChunkSize" : "128k",
      "bufferMaxSize" : "512k",
      "caCert" : "",
      "clusterID" : huaweicloud_cce_autopilot_cluster.test.id,
      "clusterName" : huaweicloud_cce_autopilot_cluster.test.name,
      "cluster_category" : "CCE",
      "createAudit" : true,
      "createDefaultEvent" : true,
      "createDefaultEventToAOM" : true,
      "createDefaultStdout" : true,
      "createKubeApiserver" : false,
      "createKubeControllerManager" : false,
      "createKubeScheduler" : false,
      "enableEventReport" : true,
      "enableFullPathCollection" : false,
      "enableGcrypto" : true,
      "enableLogOperatorHA" : true,
      "enableLogReport" : true,
      "featureGates" : ["sendAOMAllEvent", "outputKafka", "podLabelExclude", "fullPathCollection", "independentEvents"],
      "host_network" : false,
      "ltsAccessEndpoint" : "https://lts-access.${var.region_name}.myhuaweicloud.com:8102",
      "ltsAuditStreamID" : "",
      "ltsEndpoint" : "https://lts.${var.region_name}.myhuaweicloud.com",
      "ltsEnterpriseProjectID" : var.cluster_enterprise_project_id,
      "ltsEventStreamID" : "",
      "ltsGroupID" : "",
      "ltsKubeApiserverStreamID" : "",
      "ltsKubeControllerManagerStreamID" : "",
      "ltsKubeSchedulerStreamID" : "",
      "ltsLogReportDomain" : "",
      "ltsPrivateEndpointIP" : "",
      "ltsStdoutStreamID" : "",
      "maxEventAgeSeconds" : "",
      "memBufLimit" : "40mb",
      "multiAZEnabled" : false,
      "otelReportLogs" : true,
      "paasakskEnable" : true,
      "podDisruptionBudget" : { "create" : true, "maxUnavailable" : 1 },
      "projectID" : "",
      "secretKey" : "",
      "securityToken" : "",
      "serverCert" : "",
      "serverKey" : ""
    }, var.addon_values_custom))
  }

  depends_on = [
    huaweicloud_cce_autopilot_cluster.test,
    huaweicloud_swr_organization.test
  ]

  lifecycle {
    ignore_changes = [
      values
    ]
  }
}
```

**参数说明**：
- **cluster_id**：插件所属的集群ID，引用前一步创建的CCE Autopilot集群资源的ID进行赋值
- **addon_template_name**：插件模板名称，通过引用输入变量 addon_template_name 进行赋值
- **version**：插件版本，通过引用输入变量 addon_version 进行赋值；未指定时使用最新的兼容版本
- **name**：插件名称，通过引用输入变量 addon_name 进行赋值
- **alias**：插件别名，通过引用输入变量 addon_alias 进行赋值
- **values**：插件配置值，包含 basic、flavor 和 custom 三部分，分别通过引用输入变量 addon_values_basic、addon_values_flavor 和 addon_values_custom 进行合并赋值；其中 basic 部分引用了SWR组织名称和集群版本，custom 部分引用了集群ID、集群名称和企业项目ID

### 7. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# VPC和子网变量
vpc_name    = "tf_test_vpc"
subnet_name = "tf_test_subnet"

# 集群变量
cluster_name = "tf-test-autopilot-cluster"

# SWR组织变量
swr_organization_name = "tf-test-swr-org"

# 插件变量
addon_template_name = "log-agent"
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
3. 确认资源计划无误后，运行 `terraform apply` 开始创建CCE Autopilot集群及其插件
4. 运行 `terraform show` 查看已创建的CCE Autopilot集群及其插件

## 参考信息

- [华为云云容器引擎产品文档](https://support.huaweicloud.com/cce/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [CCE Autopilot集群插件最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cceautopilot/cce-autopilot-addons)
