# 部署队列公网连通

## 应用场景

数据湖探索（Data Lake Insight，DLI）的队列默认部署在用户VPC内，无法直接访问公网上的数据源或服务。当业务需要让DLI队列访问公网（例如访问外部API、拉取公网数据或与公网服务交互）时，需要为队列配置增强型跨源连接，并通过NAT网关的SNAT规则实现公网出口。

本最佳实践将介绍如何使用Terraform自动化部署DLI队列公网连通，包括弹性资源池、队列、VPC与子网、增强型跨源连接、弹性公网IP、NAT网关及SNAT规则的创建与关联。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [DLI弹性资源池（huaweicloud_dli_elastic_resource_pool）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_elastic_resource_pool)
- [DLI队列（huaweicloud_dli_queue）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_queue)
- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [VPC子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [DLI增强型跨源连接（huaweicloud_dli_datasource_connection）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_datasource_connection)
- [DLI增强型跨源连接关联（huaweicloud_dli_datasource_connection_associate）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dli_datasource_connection_associate)
- [弹性公网IP（huaweicloud_vpc_eip）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_eip)
- [NAT网关（huaweicloud_nat_gateway）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/nat_gateway)
- [NAT SNAT规则（huaweicloud_nat_snat_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/nat_snat_rule)

### 资源/数据源依赖关系

```
huaweicloud_dli_elastic_resource_pool
    ├── huaweicloud_dli_queue
    │   └── huaweicloud_dli_datasource_connection_associate
    ├── huaweicloud_dli_datasource_connection_associate
    └── huaweicloud_nat_snat_rule

huaweicloud_vpc
    ├── huaweicloud_vpc_subnet
    │   ├── huaweicloud_dli_datasource_connection
    │   └── huaweicloud_nat_gateway
    ├── huaweicloud_dli_datasource_connection
    └── huaweicloud_nat_gateway

huaweicloud_vpc_eip
    └── huaweicloud_nat_snat_rule
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建DLI弹性资源池

在TF文件（如main.tf）中添加以下脚本以创建DLI弹性资源池：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DLI弹性资源池
variable "elastic_resource_pool_name" {
  description = "The name of the DLI elastic resource pool"
  type        = string
}

variable "elastic_resource_pool_description" {
  description = "The description of the elastic resource pool"
  type        = string
  default     = ""
}

variable "elastic_resource_pool_min_cu" {
  description = "The minimum number of CUs for the elastic resource pool"
  type        = number
  default     = 16
}

variable "elastic_resource_pool_max_cu" {
  description = "The maximum number of CUs for the elastic resource pool"
  type        = number
  default     = 64
}

variable "elastic_resource_pool_cidr" {
  description = "The CIDR block of the elastic resource pool. This CIDR must not overlap with the VPC CIDR"
  type        = string
}

variable "enterprise_project_id" {
  description = "The ID of the enterprise project"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_dli_elastic_resource_pool" "test" {
  name                  = var.elastic_resource_pool_name
  description           = var.elastic_resource_pool_description
  min_cu                = var.elastic_resource_pool_min_cu
  max_cu                = var.elastic_resource_pool_max_cu
  cidr                  = var.elastic_resource_pool_cidr
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null

  label = {
    spec = "basic"
  }
}
```

**参数说明**：
- **name**：弹性资源池名称，通过引用输入变量 elastic_resource_pool_name 进行赋值
- **description**：弹性资源池描述，通过引用输入变量 elastic_resource_pool_description 进行赋值
- **min_cu**：弹性资源池的最小CU数，通过引用输入变量 elastic_resource_pool_min_cu 进行赋值
- **max_cu**：弹性资源池的最大CU数，通过引用输入变量 elastic_resource_pool_max_cu 进行赋值
- **cidr**：弹性资源池的网段，通过引用输入变量 elastic_resource_pool_cidr 进行赋值，该网段不能与VPC网段重叠
- **enterprise_project_id**：企业项目ID，通过引用输入变量 enterprise_project_id 进行赋值

### 3. 创建DLI队列

在TF文件中添加以下脚本以创建DLI队列：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DLI队列
variable "queue_name" {
  description = "The name of the DLI exclusive queue"
  type        = string
}

variable "queue_type" {
  description = "The type of the DLI queue. The valid values are sql and general"
  type        = string
  default     = "sql"

  validation {
    condition     = contains(["sql", "general"], var.queue_type)
    error_message = "The queue_type valid value must be `sql` or `general`."
  }
}

variable "queue_cu_count" {
  description = "The CU count of the DLI queue"
  type        = number
  default     = 16
}

variable "queue_description" {
  description = "The description of the DLI queue"
  type        = string
  default     = ""
}

resource "huaweicloud_dli_queue" "test" {
  elastic_resource_pool_name = huaweicloud_dli_elastic_resource_pool.test.name
  resource_mode              = 1

  name                  = var.queue_name
  queue_type            = var.queue_type
  cu_count              = var.queue_cu_count
  description           = var.queue_description
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}
```

**参数说明**：
- **elastic_resource_pool_name**：队列所属的弹性资源池名称，引用上一步创建的弹性资源池名称进行赋值
- **resource_mode**：资源模式，设置为1表示使用弹性资源池模式
- **name**：队列名称，通过引用输入变量 queue_name 进行赋值
- **queue_type**：队列类型，通过引用输入变量 queue_type 进行赋值，有效值为sql和general
- **cu_count**：队列的CU数，通过引用输入变量 queue_cu_count 进行赋值
- **description**：队列描述，通过引用输入变量 queue_description 进行赋值
- **enterprise_project_id**：企业项目ID，通过引用输入变量 enterprise_project_id 进行赋值

### 4. 创建VPC和子网

在TF文件中添加以下脚本以创建VPC和子网：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建VPC和子网
variable "vpc_name" {
  description = "The name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
}

variable "subnet_name" {
  description = "The name of the subnet"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet. If empty, it is calculated from the VPC CIDR"
  type        = string
  default     = ""
  nullable    = false
}

variable "subnet_gateway_ip" {
  description = "The gateway IP of the subnet. If empty, it is calculated from the subnet CIDR"
  type        = string
  default     = ""
  nullable    = false
}

resource "huaweicloud_vpc" "test" {
  name                  = var.vpc_name
  cidr                  = var.vpc_cidr
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr != "" ? var.subnet_cidr : cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0)
  gateway_ip = var.subnet_gateway_ip != "" ? var.subnet_gateway_ip : cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1)
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量 vpc_name 进行赋值
- **cidr**：VPC网段，通过引用输入变量 vpc_cidr 进行赋值
- **enterprise_project_id**：企业项目ID，通过引用输入变量 enterprise_project_id 进行赋值
- **vpc_id**：子网所属的VPC ID，引用上一步创建的VPC ID进行赋值
- **subnet name**：子网名称，通过引用输入变量 subnet_name 进行赋值
- **subnet cidr**：子网网段，通过引用输入变量 subnet_cidr 进行赋值，若为空则根据VPC网段自动计算
- **gateway_ip**：子网网关IP，通过引用输入变量 subnet_gateway_ip 进行赋值，若为空则根据子网网段自动计算

### 5. 创建DLI增强型跨源连接

在TF文件中添加以下脚本以创建DLI增强型跨源连接：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DLI增强型跨源连接
variable "datasource_connection_name" {
  description = "The name of the DLI enhanced datasource connection"
  type        = string
}

variable "datasource_connection_hosts" {
  description = "The list of custom hosts for the enhanced datasource connection"

  type = list(object({
    name = string
    ip   = string
  }))

  default  = []
  nullable = false
}

variable "datasource_connection_routes" {
  description = "The list of custom routes for the enhanced datasource connection. Each cidr should be the public destination network to access"

  type = list(object({
    name = string
    cidr = string
  }))
}

resource "huaweicloud_dli_datasource_connection" "test" {
  name      = var.datasource_connection_name
  vpc_id    = huaweicloud_vpc.test.id
  subnet_id = huaweicloud_vpc_subnet.test.id

  dynamic "hosts" {
    for_each = var.datasource_connection_hosts

    content {
      name = hosts.value.name
      ip   = hosts.value.ip
    }
  }

  dynamic "routes" {
    for_each = var.datasource_connection_routes

    content {
      name = routes.value.name
      cidr = routes.value.cidr
    }
  }
}
```

**参数说明**：
- **name**：增强型跨源连接名称，通过引用输入变量 datasource_connection_name 进行赋值
- **vpc_id**：跨源连接所属的VPC ID，引用上一步创建的VPC ID进行赋值
- **subnet_id**：跨源连接所属的子网ID，引用上一步创建的子网ID进行赋值
- **hosts**：自定义主机信息，通过引用输入变量 datasource_connection_hosts 进行赋值
- **routes**：自定义路由信息，通过引用输入变量 datasource_connection_routes 进行赋值，其中cidr为需要访问的公网目的网段

### 6. 关联增强型跨源连接与弹性资源池

在TF文件中添加以下脚本以将增强型跨源连接关联到弹性资源池：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下关联增强型跨源连接与弹性资源池
resource "huaweicloud_dli_datasource_connection_associate" "test" {
  connection_id          = huaweicloud_dli_datasource_connection.test.id
  elastic_resource_pools = [huaweicloud_dli_elastic_resource_pool.test.name]

  depends_on = [huaweicloud_dli_queue.test]
}
```

**参数说明**：
- **connection_id**：增强型跨源连接ID，引用上一步创建的增强型跨源连接ID进行赋值
- **elastic_resource_pools**：需要关联的弹性资源池名称列表，引用上一步创建的弹性资源池名称进行赋值
- **depends_on**：显式依赖关系，确保队列创建完成后再执行关联操作

### 7. 创建弹性公网IP

在TF文件中添加以下脚本以创建弹性公网IP：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建弹性公网IP
variable "eip_type" {
  description = "The type of the EIP"
  type        = string
  default     = "5_bgp"
}

variable "eip_bandwidth_name" {
  description = "The name of the EIP bandwidth"
  type        = string
}

variable "eip_bandwidth_size" {
  description = "The size of the EIP bandwidth in Mbps"
  type        = number
  default     = 5
}

variable "eip_bandwidth_share_type" {
  description = "The share type of the EIP bandwidth"
  type        = string
  default     = "PER"
}

variable "eip_bandwidth_charge_mode" {
  description = "The charge mode of the EIP bandwidth"
  type        = string
  default     = "traffic"
}

resource "huaweicloud_vpc_eip" "test" {
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null

  publicip {
    type = var.eip_type
  }

  bandwidth {
    name        = var.eip_bandwidth_name
    size        = var.eip_bandwidth_size
    share_type  = var.eip_bandwidth_share_type
    charge_mode = var.eip_bandwidth_charge_mode
  }
}
```

**参数说明**：
- **enterprise_project_id**：企业项目ID，通过引用输入变量 enterprise_project_id 进行赋值
- **publicip.type**：弹性公网IP类型，通过引用输入变量 eip_type 进行赋值
- **bandwidth.name**：带宽名称，通过引用输入变量 eip_bandwidth_name 进行赋值
- **bandwidth.size**：带宽大小，通过引用输入变量 eip_bandwidth_size 进行赋值
- **bandwidth.share_type**：带宽共享类型，通过引用输入变量 eip_bandwidth_share_type 进行赋值
- **bandwidth.charge_mode**：带宽计费模式，通过引用输入变量 eip_bandwidth_charge_mode 进行赋值

### 8. 创建NAT网关

在TF文件中添加以下脚本以创建NAT网关：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建NAT网关
variable "nat_gateway_name" {
  description = "The name of the NAT gateway"
  type        = string
}

variable "nat_gateway_spec" {
  description = "The specification of the NAT gateway."
  type        = string
  default     = "1"

  validation {
    condition     = contains(["1", "2", "3", "4"], var.nat_gateway_spec)
    error_message = "The nat_gateway_spec valid value must be `1`, `2`, `3` or `4`."
  }
}

variable "nat_gateway_description" {
  description = "The description of the NAT gateway"
  type        = string
  default     = ""
}

resource "huaweicloud_nat_gateway" "test" {
  name                  = var.nat_gateway_name
  spec                  = var.nat_gateway_spec
  vpc_id                = huaweicloud_vpc.test.id
  subnet_id             = huaweicloud_vpc_subnet.test.id
  description           = var.nat_gateway_description
  enterprise_project_id = var.enterprise_project_id != "" ? var.enterprise_project_id : null
}
```

**参数说明**：
- **name**：NAT网关名称，通过引用输入变量 nat_gateway_name 进行赋值
- **spec**：NAT网关规格，通过引用输入变量 nat_gateway_spec 进行赋值，有效值为1、2、3、4
- **vpc_id**：NAT网关所属的VPC ID，引用上一步创建的VPC ID进行赋值
- **subnet_id**：NAT网关所属的子网ID，引用上一步创建的子网ID进行赋值
- **description**：NAT网关描述，通过引用输入变量 nat_gateway_description 进行赋值
- **enterprise_project_id**：企业项目ID，通过引用输入变量 enterprise_project_id 进行赋值

### 9. 创建SNAT规则

在TF文件中添加以下脚本以创建SNAT规则：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建SNAT规则
variable "snat_description" {
  description = "The description of the SNAT rule"
  type        = string
  default     = ""
}

resource "huaweicloud_nat_snat_rule" "test" {
  nat_gateway_id = huaweicloud_nat_gateway.test.id
  floating_ip_id = huaweicloud_vpc_eip.test.id
  source_type    = 1
  cidr           = huaweicloud_dli_elastic_resource_pool.test.cidr
  description    = var.snat_description
}
```

**参数说明**：
- **nat_gateway_id**：NAT网关ID，引用上一步创建的NAT网关ID进行赋值
- **floating_ip_id**：弹性公网IP ID，引用上一步创建的弹性公网IP ID进行赋值
- **source_type**：源类型，设置为1表示源网段
- **cidr**：源网段，引用弹性资源池的网段进行赋值，使弹性资源池内的队列可以通过SNAT访问公网
- **description**：SNAT规则描述，通过引用输入变量 snat_description 进行赋值

### 10. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
vpc_name                   = "tf_test_dli_vpc"
vpc_cidr                   = "192.168.0.0/16"
subnet_name                = "tf_test_dli_subnet"
elastic_resource_pool_name = "tf_test_dli_pool"
elastic_resource_pool_cidr = "172.16.0.0/18"
queue_name                 = "tf_test_dli_queue"
datasource_connection_name = "tf_test_dli_conn"

datasource_connection_routes = [
  {
    name = "tf_test_dli_route"
    cidr = "14.17.72.0/24"
  }
]

eip_bandwidth_name = "tf_test_dli_eip_bw"
nat_gateway_name   = "tf_test_dli_nat"
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

### 11. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DLI队列公网连通相关资源
4. 运行 `terraform show` 查看已创建的DLI队列公网连通相关资源

## 参考信息

- [华为云数据湖探索产品文档](https://support.huaweicloud.com/dli/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DLI队列公网连通最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dli/queue-public-network-connectivity)
