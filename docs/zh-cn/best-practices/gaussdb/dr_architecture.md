# 部署容灾架构

## 应用场景

GaussDB是华为云提供的高性能、高可用、高安全的企业级分布式关系型数据库服务，支持集中式和分布式两种部署形态。在核心业务系统中，单实例部署难以满足业务连续性要求，需要构建跨可用区甚至跨区域的容灾架构，在主实例发生故障时能够快速切换，保障数据不丢失、业务不中断。

本最佳实践将介绍如何使用Terraform自动化部署GaussDB容灾架构，包括主备实例所在区域的VPC、子网与安全组创建，云连接（CC）跨区域网络打通，主备GaussDB实例创建，容灾配置重置以及容灾关系建立。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [IAM项目（data.huaweicloud_identity_projects）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/identity_projects)

### 资源

- [虚拟私有云（huaweicloud_vpc）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [虚拟私有云子网（huaweicloud_vpc_subnet）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [安全组（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)
- [云连接（huaweicloud_cc_connection）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cc_connection)
- [云连接网络实例（huaweicloud_cc_network_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cc_network_instance)
- [云连接带宽包（huaweicloud_cc_bandwidth_package）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cc_bandwidth_package)
- [云连接域间带宽（huaweicloud_cc_inter_region_bandwidth）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cc_inter_region_bandwidth)
- [随机密码（random_password）](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)
- [GaussDB实例（huaweicloud_gaussdb_instance）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_instance)
- [GaussDB容灾配置重置（huaweicloud_gaussdb_dr_configuration_reset）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_dr_configuration_reset)
- [GaussDB容灾关系（huaweicloud_gaussdb_dr_relationship）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/gaussdb_dr_relationship)

### 资源/数据源依赖关系

```
data.huaweicloud_identity_projects
    ├── huaweicloud_cc_network_instance.primary
    └── huaweicloud_cc_network_instance.dr

huaweicloud_vpc.primary
    ├── huaweicloud_vpc_subnet.primary
    ├── huaweicloud_networking_secgroup.primary
    └── huaweicloud_cc_network_instance.primary

huaweicloud_vpc.dr
    ├── huaweicloud_vpc_subnet.dr
    ├── huaweicloud_networking_secgroup.dr
    └── huaweicloud_cc_network_instance.dr

huaweicloud_cc_connection.dr
    ├── huaweicloud_cc_network_instance.primary
    ├── huaweicloud_cc_network_instance.dr
    ├── huaweicloud_cc_bandwidth_package.dr
    └── huaweicloud_cc_inter_region_bandwidth.dr

huaweicloud_gaussdb_instance.primary
    ├── huaweicloud_gaussdb_dr_configuration_reset.primary
    └── huaweicloud_gaussdb_dr_relationship.test

huaweicloud_gaussdb_instance.dr
    └── huaweicloud_gaussdb_dr_configuration_reset.dr
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建主区域网络资源

在TF文件（如main.tf）中添加以下脚本以创建主区域的VPC、子网、安全组及安全组规则：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建主区域的VPC资源
variable "primary_vpc_name" {
  description = "The VPC name in the primary region"
  type        = string
}

variable "primary_vpc_cidr" {
  description = "The CIDR block of the VPC in the primary region"
  type        = string
  default     = "172.16.0.0/16"
}

variable "primary_subnet_names" {
  description = "The subnet names for each AZ in the primary region"
  type        = list(string)
}

variable "primary_availability_zones" {
  description = "The availability zones in the primary region"
  type        = list(string)
}

variable "primary_security_group_name" {
  description = "The security group name in the primary region"
  type        = string
}

variable "secgroup_rules" {
  description = "The security group ingress rules"
  type        = list(object({
    ports  = string
    source = string
  }))
  default     = [
    { ports = "5432-5532", source = "local" },
    { ports = "5432-5532", source = "remote" },
    { ports = "20050",     source = "local" },
    { ports = "20050",     source = "remote" },
    { ports = "5000-5001", source = "local" },
    { ports = "5000-5001", source = "remote" },
    { ports = "2379-2380", source = "local" },
    { ports = "2379-2380", source = "remote" },
    { ports = "6000",      source = "local" },
    { ports = "6000",      source = "remote" },
    { ports = "6500",      source = "local" },
    { ports = "6500",      source = "remote" },
    { ports = "12016",     source = "local" },
    { ports = "12016",     source = "remote" },
  ]
}

resource "huaweicloud_vpc" "primary" {
  name = var.primary_vpc_name
  cidr = var.primary_vpc_cidr
}

resource "huaweicloud_vpc_subnet" "primary" {
  count = length(var.primary_subnet_names)

  vpc_id            = huaweicloud_vpc.primary.id
  name              = try(var.primary_subnet_names[count.index], null)
  cidr              = cidrsubnet(huaweicloud_vpc.primary.cidr, 8, count.index)
  gateway_ip        = cidrhost(cidrsubnet(huaweicloud_vpc.primary.cidr, 8, count.index), 1)
  availability_zone = try(var.primary_availability_zones[count.index], null)
}

resource "huaweicloud_networking_secgroup" "primary" {
  name                 = var.primary_security_group_name
  delete_default_rules = true
}

resource "huaweicloud_networking_secgroup_rule" "primary" {
  count = length(var.secgroup_rules)

  security_group_id = huaweicloud_networking_secgroup.primary.id
  direction         = "ingress"
  ethertype         = "IPv4"
  remote_ip_prefix  = var.secgroup_rules[count.index].source == "local" ? huaweicloud_vpc.primary.cidr : huaweicloud_vpc.dr.cidr
  ports             = try(var.secgroup_rules[count.index].ports, null)
  protocol          = "tcp"
}
```

**参数说明**：
- **name**：VPC名称，通过引用输入变量 primary_vpc_name 进行赋值
- **cidr**：VPC网段，通过引用输入变量 primary_vpc_cidr 进行赋值
- **vpc_id**：子网所属的VPC ID，引用 `huaweicloud_vpc.primary.id`
- **availability_zone**：子网所在的可用区，通过引用输入变量 primary_availability_zones 进行赋值
- **security_group_id**：安全组规则所属的安全组 ID，引用 `huaweicloud_networking_secgroup.primary.id`
- **remote_ip_prefix**：安全组规则的远端IP地址段，根据规则来源引用主区域或容灾区域的VPC网段
- **ports**：安全组规则开放的端口范围，通过引用输入变量 secgroup_rules 进行赋值

### 3. 创建容灾区域网络资源

在TF文件（如main.tf）中添加以下脚本以创建容灾区域的VPC、子网、安全组及安全组规则：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建容灾区域的VPC资源
variable "dr_vpc_name" {
  description = "The VPC name in the DR region"
  type        = string
}

variable "dr_vpc_cidr" {
  description = "The CIDR block of the VPC in the DR region"
  type        = string
  default     = "172.17.0.0/16"
}

variable "dr_subnet_name" {
  description = "The subnet name in the DR region"
  type        = string
}

variable "dr_availability_zone" {
  description = "The availability zone in the DR region"
  type        = string
}

variable "dr_security_group_name" {
  description = "The security group name in the DR region"
  type        = string
}

resource "huaweicloud_vpc" "dr" {
  provider = huaweicloud.dr

  name = var.dr_vpc_name
  cidr = var.dr_vpc_cidr
}

resource "huaweicloud_vpc_subnet" "dr" {
  provider = huaweicloud.dr

  vpc_id            = huaweicloud_vpc.dr.id
  name              = var.dr_subnet_name
  cidr              = cidrsubnet(huaweicloud_vpc.dr.cidr, 8, 0)
  gateway_ip        = cidrhost(cidrsubnet(huaweicloud_vpc.dr.cidr, 8, 0), 1)
  availability_zone = var.dr_availability_zone
}

resource "huaweicloud_networking_secgroup" "dr" {
  provider = huaweicloud.dr

  name                 = var.dr_security_group_name
  delete_default_rules = true
}

resource "huaweicloud_networking_secgroup_rule" "dr" {
  provider = huaweicloud.dr

  count = length(var.secgroup_rules)

  security_group_id = huaweicloud_networking_secgroup.dr.id
  direction         = "ingress"
  ethertype         = "IPv4"
  remote_ip_prefix  = var.secgroup_rules[count.index].source == "local" ? huaweicloud_vpc.dr.cidr : huaweicloud_vpc.primary.cidr
  ports             = try(var.secgroup_rules[count.index].ports, null)
  protocol          = "tcp"
}
```

**参数说明**：
- **provider**：指定使用别名为 `huaweicloud.dr` 的provider，将资源创建在容灾区域
- **name**：VPC名称，通过引用输入变量 dr_vpc_name 进行赋值
- **cidr**：VPC网段，通过引用输入变量 dr_vpc_cidr 进行赋值
- **vpc_id**：子网所属的VPC ID，引用 `huaweicloud_vpc.dr.id`
- **availability_zone**：子网所在的可用区，通过引用输入变量 dr_availability_zone 进行赋值
- **security_group_id**：安全组规则所属的安全组 ID，引用 `huaweicloud_networking_secgroup.dr.id`
- **remote_ip_prefix**：安全组规则的远端IP地址段，根据规则来源引用容灾区域或主区域的VPC网段

### 4. 创建云连接网络实例

在TF文件（如main.tf）中添加以下脚本以查询IAM项目并创建云连接及网络实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下查询IAM项目信息
data "huaweicloud_identity_projects" "primary" {
  name = var.region_name
}

data "huaweicloud_identity_projects" "dr" {
  provider = huaweicloud.dr

  name = var.dr_region_name
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云连接资源
resource "huaweicloud_cc_connection" "dr" {
  name        = "${var.primary_vpc_name}_cc"
  description = "Cloud connection for GaussDB cross-region DR"
}

resource "huaweicloud_cc_network_instance" "primary" {
  cloud_connection_id = huaweicloud_cc_connection.dr.id
  instance_id         = huaweicloud_vpc.primary.id
  project_id          = try(data.huaweicloud_identity_projects.primary.projects[0].id, null)
  region_id           = var.region_name
  type                = "vpc"
  cidrs               = [huaweicloud_vpc.primary.cidr]
}

resource "huaweicloud_cc_network_instance" "dr" {
  provider = huaweicloud.dr

  cloud_connection_id = huaweicloud_cc_connection.dr.id
  instance_id         = huaweicloud_vpc.dr.id
  project_id          = try(data.huaweicloud_identity_projects.dr.projects[0].id, null)
  region_id           = var.dr_region_name
  type                = "vpc"
  cidrs               = [huaweicloud_vpc.dr.cidr]
}
```

**参数说明**：
- **name**：IAM项目名称，通过引用输入变量 region_name 或 dr_region_name 进行赋值
- **cloud_connection_id**：网络实例所属的云连接 ID，引用 `huaweicloud_cc_connection.dr.id`
- **instance_id**：网络实例关联的VPC ID，引用 `huaweicloud_vpc.primary.id` 或 `huaweicloud_vpc.dr.id`
- **project_id**：网络实例所属的IAM项目 ID，引用数据源查询结果
- **region_id**：网络实例所在的区域，通过引用输入变量 region_name 或 dr_region_name 进行赋值
- **type**：网络实例类型，此处为 `vpc`
- **cidrs**：网络实例的网段，引用对应VPC的网段

### 5. 创建云连接带宽包与域间带宽

在TF文件（如main.tf）中添加以下脚本以创建云连接带宽包和域间带宽：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建云连接带宽包资源
variable "cc_bandwidth" {
  description = "The inter-region bandwidth (Mbit/s) for the Cloud Connection"
  type        = number
  default     = 10
}

resource "huaweicloud_cc_bandwidth_package" "dr" {
  name           = "${var.primary_vpc_name}_cc_bp"
  local_area_id  = "Chinese-Mainland"
  remote_area_id = "Chinese-Mainland"
  bandwidth      = var.cc_bandwidth
  billing_mode   = 3
  charge_mode    = "bandwidth"
  resource_id    = huaweicloud_cc_connection.dr.id
  resource_type  = "cloud_connection"
}

resource "huaweicloud_cc_inter_region_bandwidth" "dr" {
  cloud_connection_id  = huaweicloud_cc_connection.dr.id
  bandwidth_package_id = huaweicloud_cc_bandwidth_package.dr.id
  bandwidth            = var.cc_bandwidth
  inter_region_ids     = [var.region_name, var.dr_region_name]
}
```

**参数说明**：
- **name**：带宽包名称，由主区域VPC名称拼接生成
- **local_area_id**：带宽包本端区域，此处为 `Chinese-Mainland`
- **remote_area_id**：带宽包对端区域，此处为 `Chinese-Mainland`
- **bandwidth**：带宽大小，通过引用输入变量 cc_bandwidth 进行赋值
- **billing_mode**：计费模式，此处为 `3`
- **charge_mode**：计费方式，此处为 `bandwidth`
- **resource_id**：带宽包关联的云连接 ID，引用 `huaweicloud_cc_connection.dr.id`
- **resource_type**：带宽包关联的资源类型，此处为 `cloud_connection`
- **cloud_connection_id**：域间带宽所属的云连接 ID，引用 `huaweicloud_cc_connection.dr.id`
- **bandwidth_package_id**：域间带宽关联的带宽包 ID，引用 `huaweicloud_cc_bandwidth_package.dr.id`
- **inter_region_ids**：域间带宽连接的两个区域，通过引用输入变量 region_name 和 dr_region_name 进行赋值

### 6. 创建GaussDB实例密码

在TF文件（如main.tf）中添加以下脚本以创建GaussDB实例所需的随机密码：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建随机密码资源
variable "instance_passwords" {
  description = "The passwords for GaussDB instances (empty string to auto-generate), index 0=primary, 1=dr"
  type        = list(string)
  default     = ["", ""]
  sensitive   = true
}

resource "random_password" "instance" {
  count = length(var.instance_passwords)

  length           = 12
  special          = true
  override_special = "!@%^*-_=+"

  keepers = {
    password_needed = try(var.instance_passwords[count.index], "") == "" ? "true" : "false"
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

**参数说明**：
- **count**：随机密码数量，与输入变量 instance_passwords 的长度一致
- **length**：密码长度，此处为 `12`
- **special**：是否包含特殊字符，此处为 `true`
- **override_special**：允许使用的特殊字符集合
- **keepers**：用于触发密码重新生成的键值对，当输入变量 instance_passwords 中对应元素为空字符串时生成新密码

### 7. 创建主区域GaussDB实例

在TF文件（如main.tf）中添加以下脚本以创建主区域的GaussDB实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建主区域的GaussDB实例资源
variable "primary_instance_name" {
  description = "The name of the primary GaussDB instance"
  type        = string
}

variable "instance_flavor" {
  description = "The spec_code of the GaussDB instance flavor"
  type        = string
}

variable "primary_instance_availability_zones" {
  description = "The comma-separated AZ string for the primary GaussDB instance"
  type        = string
}

variable "instance_db_port" {
  description = "The database port"
  type        = number
  default     = 5432
}

variable "enterprise_project_id" {
  description = "The enterprise project ID to which the GaussDB instances belong"
  type        = string
}

variable "primary_instance_volume_type" {
  description = "The storage volume type of the primary GaussDB instance"
  type        = string
  default     = "ULTRAHIGH"
}

variable "primary_instance_volume_size" {
  description = "The storage volume size (GB) of the primary GaussDB instance"
  type        = number
  default     = 40
}

resource "huaweicloud_gaussdb_instance" "primary" {
  name                  = var.primary_instance_name
  flavor                = var.instance_flavor
  password              = try(random_password.instance[0].result, null)
  vpc_id                = huaweicloud_vpc.primary.id
  subnet_id             = try(huaweicloud_vpc_subnet.primary[0].id, null)
  security_group_id     = huaweicloud_networking_secgroup.primary.id
  availability_zone     = var.primary_instance_availability_zones
  port                  = var.instance_db_port
  enterprise_project_id = var.enterprise_project_id

  ha {
    mode             = "centralization_standard"
    replication_mode = "sync"
    consistency      = "strong"
  }

  replica_num = 3

  volume {
    type = var.primary_instance_volume_type
    size = var.primary_instance_volume_size
  }

  lifecycle {
    ignore_changes = [
      flavor,
    ]
  }
}
```

**参数说明**：
- **name**：GaussDB实例名称，通过引用输入变量 primary_instance_name 进行赋值
- **flavor**：实例规格，通过引用输入变量 instance_flavor 进行赋值
- **password**：实例密码，引用随机密码资源生成结果
- **vpc_id**：实例所属的VPC ID，引用 `huaweicloud_vpc.primary.id`
- **subnet_id**：实例所属的子网 ID，引用 `huaweicloud_vpc_subnet.primary[0].id`
- **security_group_id**：实例所属的安全组 ID，引用 `huaweicloud_networking_secgroup.primary.id`
- **availability_zone**：实例所在的可用区，通过引用输入变量 primary_instance_availability_zones 进行赋值
- **port**：数据库端口，通过引用输入变量 instance_db_port 进行赋值
- **enterprise_project_id**：企业项目 ID，通过引用输入变量 enterprise_project_id 进行赋值
- **ha.mode**：高可用模式，此处为 `centralization_standard`
- **ha.replication_mode**：复制模式，此处为 `sync`
- **ha.consistency**：一致性模式，此处为 `strong`
- **replica_num**：副本数量，此处为 `3`
- **volume.type**：存储类型，通过引用输入变量 primary_instance_volume_type 进行赋值
- **volume.size**：存储大小，通过引用输入变量 primary_instance_volume_size 进行赋值

### 8. 创建容灾区域GaussDB实例

在TF文件（如main.tf）中添加以下脚本以创建容灾区域的GaussDB实例：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建容灾区域的GaussDB实例资源
variable "dr_instance_name" {
  description = "The name of the DR GaussDB instance"
  type        = string
}

variable "dr_instance_availability_zones" {
  description = "The comma-separated AZ string for the DR GaussDB instance"
  type        = string
}

variable "dr_instance_volume_type" {
  description = "The storage volume type of the DR GaussDB instance"
  type        = string
  default     = "ULTRAHIGH"
}

variable "dr_instance_volume_size" {
  description = "The storage volume size (GB) of the DR GaussDB instance"
  type        = number
  default     = 40
}

resource "huaweicloud_gaussdb_instance" "dr" {
  provider = huaweicloud.dr

  name                  = var.dr_instance_name
  flavor                = var.instance_flavor
  password              = try(random_password.instance[1].result, null)
  vpc_id                = huaweicloud_vpc.dr.id
  subnet_id             = huaweicloud_vpc_subnet.dr.id
  security_group_id     = huaweicloud_networking_secgroup.dr.id
  availability_zone     = var.dr_instance_availability_zones
  port                  = var.instance_db_port
  enterprise_project_id = var.enterprise_project_id

  ha {
    mode             = "centralization_standard"
    replication_mode = "sync"
    consistency      = "strong"
  }

  replica_num = 3

  volume {
    type = var.dr_instance_volume_type
    size = var.dr_instance_volume_size
  }

  lifecycle {
    ignore_changes = [
      flavor,
    ]
  }
}
```

**参数说明**：
- **provider**：指定使用别名为 `huaweicloud.dr` 的provider，将实例创建在容灾区域
- **name**：GaussDB实例名称，通过引用输入变量 dr_instance_name 进行赋值
- **flavor**：实例规格，通过引用输入变量 instance_flavor 进行赋值
- **password**：实例密码，引用随机密码资源生成结果
- **vpc_id**：实例所属的VPC ID，引用 `huaweicloud_vpc.dr.id`
- **subnet_id**：实例所属的子网 ID，引用 `huaweicloud_vpc_subnet.dr.id`
- **security_group_id**：实例所属的安全组 ID，引用 `huaweicloud_networking_secgroup.dr.id`
- **availability_zone**：实例所在的可用区，通过引用输入变量 dr_instance_availability_zones 进行赋值
- **port**：数据库端口，通过引用输入变量 instance_db_port 进行赋值
- **enterprise_project_id**：企业项目 ID，通过引用输入变量 enterprise_project_id 进行赋值
- **volume.type**：存储类型，通过引用输入变量 dr_instance_volume_type 进行赋值
- **volume.size**：存储大小，通过引用输入变量 dr_instance_volume_size 进行赋值

### 9. 重置容灾配置

在TF文件（如main.tf）中添加以下脚本以重置主备实例的容灾配置：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下重置主实例的容灾配置
resource "huaweicloud_gaussdb_dr_configuration_reset" "primary" {
  instance_id        = huaweicloud_gaussdb_instance.primary.id
  opposite_data_cidr = huaweicloud_vpc_subnet.dr.cidr
}

resource "huaweicloud_gaussdb_dr_configuration_reset" "dr" {
  provider = huaweicloud.dr

  instance_id        = huaweicloud_gaussdb_instance.dr.id
  opposite_data_cidr = try(huaweicloud_vpc_subnet.primary[0].cidr, null)
}
```

**参数说明**：
- **instance_id**：需要重置容灾配置的GaussDB实例 ID，引用 `huaweicloud_gaussdb_instance.primary.id` 或 `huaweicloud_gaussdb_instance.dr.id`
- **opposite_data_cidr**：对端数据网段，主实例引用容灾区域子网网段，容灾实例引用主区域子网网段

### 10. 建立容灾关系

在TF文件（如main.tf）中添加以下脚本以建立主备实例之间的容灾关系：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下建立GaussDB容灾关系
variable "dr_disaster_type" {
  description = "The disaster recovery type"
  type        = string
  default     = "stream"
}

variable "dr_user_name" {
  description = "The database username for disaster recovery"
  type        = string
}

variable "dr_user_password" {
  description = "The database password for disaster recovery"
  type        = string
  sensitive   = true
}

resource "huaweicloud_gaussdb_dr_relationship" "test" {
  instance_id      = huaweicloud_gaussdb_instance.primary.id
  disaster_type    = var.dr_disaster_type
  dr_ip            = try(huaweicloud_gaussdb_instance.dr.private_ips[0], null)
  dr_user_name     = var.dr_user_name
  dr_user_password = var.dr_user_password

  depends_on = [
    huaweicloud_gaussdb_dr_configuration_reset.primary,
    huaweicloud_gaussdb_dr_configuration_reset.dr,
    huaweicloud_cc_network_instance.primary,
    huaweicloud_cc_network_instance.dr,
    huaweicloud_cc_inter_region_bandwidth.dr,
  ]
}
```

**参数说明**：
- **instance_id**：主实例 ID，引用 `huaweicloud_gaussdb_instance.primary.id`
- **disaster_type**：容灾类型，通过引用输入变量 dr_disaster_type 进行赋值
- **dr_ip**：容灾实例的私有IP地址，引用 `huaweicloud_gaussdb_instance.dr.private_ips[0]`
- **dr_user_name**：容灾数据库用户名，通过引用输入变量 dr_user_name 进行赋值
- **dr_user_password**：容灾数据库密码，通过引用输入变量 dr_user_password 进行赋值
- **depends_on**：显式声明依赖关系，确保容灾配置重置和云连接网络就绪后再建立容灾关系

### 11. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 主区域网络配置
primary_vpc_name            = "example_vpc"
primary_vpc_cidr            = "172.16.0.0/16"
primary_subnet_names        = ["example_subnet_1", "example_subnet_2"]
primary_availability_zones  = ["primary_region_az_1", "primary_region_az_2"]
primary_security_group_name = "example_security_group"

# 容灾区域网络配置
dr_vpc_name            = "example_vpc_dr"
dr_vpc_cidr            = "172.17.0.0/16"
dr_subnet_name         = "example_subnet_dr"
dr_availability_zone   = "dr_region_az"
dr_security_group_name = "example_security_group"
dr_region_name         = "dr_region"

# 云连接配置
cc_bandwidth = 10

# 主区域GaussDB实例配置
instance_passwords                  = ["password_1", "password_2"]
primary_instance_name               = "example_primary_instance"
instance_flavor                     = "gaussdb.opengauss.ee.c3.xlarge.x864.ha"
primary_instance_availability_zones = "primary_region_az_1,primary_region_az_2,primary_region_az_3"
instance_db_port                    = 5432
enterprise_project_id               = "your_enterprise_project_id"
primary_instance_volume_type        = "ULTRAHIGH"
primary_instance_volume_size        = 40

# 容灾区域GaussDB实例配置
dr_instance_name               = "example_dr_instance"
dr_instance_availability_zones = "dr_region_az_1,dr_region_az_2,dr_region_az_3"
dr_instance_volume_type        = "ULTRAHIGH"
dr_instance_volume_size        = 40

# 容灾关系配置
dr_user_name     = "root"
dr_user_password = "your_dr_user_password"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="primary_vpc_name=my-vpc"`
2. 环境变量：`export TF_VAR_primary_vpc_name=my-vpc`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 12. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建GaussDB容灾架构
4. 运行 `terraform show` 查看已创建的GaussDB容灾架构

## 参考信息

- [华为云GaussDB产品文档](https://support.huaweicloud.com/gaussdb/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [GaussDB容灾架构最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/gaussdb/dr-architecture)
