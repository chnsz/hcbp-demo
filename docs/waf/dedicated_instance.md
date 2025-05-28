# 使用Terraform部署WAF专业版实例

## 概述

华为云Web应用防火墙（WAF）专业版提供独享模式的Web安全防护服务，可以有效防御SQL注入、XSS跨站脚本、网页木马上传、命令注入等多种攻击。本最佳实践将介绍如何使用Terraform自动化部署WAF专业版实例。

### 应用场景

- 企业需要专属的WAF实例进行Web安全防护
- 对Web应用安全防护有高性能和定制化需求
- 需要独立部署和管理WAF实例
- 对合规性和数据隔离有严格要求

### 方案优势

- 自动化部署：使用Terraform实现基础设施即代码
- 独享资源：专属实例，资源隔离
- 高性能：独立部署，性能有保障
- 灵活配置：支持自定义配置和策略
- 合规性：满足数据安全和合规要求

### 涉及服务

- API网关（APIG）：提供API的访问控制和管理
- 统一身份认证服务（IAM）：提供身份认证和权限管理
- 虚拟私有云（VPC）：提供隔离的网络环境
- Web应用防火墙（WAF）：提供Web安全防护服务

## 资源/数据源设计

本最佳实践涉及以下主要资源和数据源：

### 数据源

1. **可用区（data.huaweicloud_availability_zones）**
   - 用途：获取当前provider块中所指定region下的所有可用区信息

### 资源

1. **VPC网络（huaweicloud_vpc）**
   - 用途：为WAF实例提供网络环境

2. **VPC子网（huaweicloud_vpc_subnet）**
   - 用途：在VPC中划分子网空间

3. **安全组（huaweicloud_networking_secgroup）**
   - 用途：控制WAF实例的网络访问

4. **安全组规则（huaweicloud_networking_secgroup_rule）**
   - 用途：配置WAF实例的访问控制规则

5. **弹性公网IP（huaweicloud_vpc_eip）**
   - 用途：为WAF实例提供公网访问能力

6. **WAF专业版实例（huaweicloud_waf_dedicated_instance）**
   - 用途：提供Web应用防护服务

### 资源/数据源依赖关系

```
data.huaweicloud_availability_zones
    └── huaweicloud_waf_dedicated_instance

huaweicloud_vpc
    └── huaweicloud_vpc_subnet
        └── huaweicloud_waf_dedicated_instance

huaweicloud_networking_secgroup
    └── huaweicloud_networking_secgroup_rule
        └── huaweicloud_waf_dedicated_instance

huaweicloud_vpc_eip
    └── huaweicloud_waf_dedicated_instance
```

## 详细配置

### 数据源配置

#### 1. 可用区（data.huaweicloud_availability_zones）

获取默认region（默认继承当前provider块中所指定的region）下所有的可用区信息，用于创建WAF专业版实例。

```hcl
data "huaweicloud_availability_zones" "test" {}
```

### 资源配置

#### 1. VPC网络（huaweicloud_vpc）

在默认region（默认继承当前provider块中所指定的region）下创建VPC网络，用于为WAF实例提供网络隔离。

```hcl
variable "vpc_name" {
  description = "VPC名称"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC的CIDR块"
  type        = string
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**参数说明**：
- **name**：VPC名称
- **cidr**：VPC网段，格式为CIDR

#### 2. VPC子网（huaweicloud_vpc_subnet）

在默认region（默认继承当前provider块中所指定的region）下的VPC网络中创建子网，用于为WAF实例提供网络空间。

```hcl
variable "subnet_name" {
  description = "子网名称"
  type        = string
}

variable "subnet_cidr" {
  description = "子网的CIDR块"
  type        = string
}

variable "subnet_gateway" {
  description = "子网的网关地址"
  type        = string
}

resource "huaweicloud_vpc_subnet" "test" {
  name       = var.subnet_name
  cidr       = var.subnet_cidr
  gateway_ip = var.subnet_gateway
  vpc_id     = huaweicloud_vpc.test.id
}
```

**参数说明**：
- **name**：子网名称
- **cidr**：子网网段，格式为CIDR
- **gateway_ip**：网关IP地址
- **vpc_id**：VPC ID

#### 3. 安全组（huaweicloud_networking_secgroup）

在默认region（默认继承当前provider块中所指定的region）下创建安全组，用于控制WAF实例的网络访问。

```hcl
variable "secgroup_name" {
  description = "安全组名称"
  type        = string
}

resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.secgroup_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组名称
- **delete_default_rules**：是否删除默认规则

#### 4. 安全组规则（huaweicloud_networking_secgroup_rule）

在默认region（默认继承当前provider块中所指定的region）下配置WAF实例的访问控制规则。

```hcl
resource "huaweicloud_networking_secgroup_rule" "allow_web" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "ingress"
  ethertype        = "IPv4"
  protocol         = "tcp"
  port_range_min   = 80
  port_range_max   = 80
  remote_ip_prefix = "0.0.0.0/0"
  description      = "允许HTTP访问"
}
```

**参数说明**：
- **security_group_id**：安全组ID
- **direction**：规则方向
- **ethertype**：网络协议版本
- **protocol**：协议类型
- **port_range_min**：起始端口
- **port_range_max**：结束端口
- **remote_ip_prefix**：允许访问的IP范围
- **description**：规则描述

#### 5. 弹性公网IP（huaweicloud_vpc_eip）

在默认region（默认继承当前provider块中所指定的region）下创建弹性公网IP，用于为WAF实例提供公网访问能力。

```hcl
variable "bandwidth_name" {
  description = "带宽名称"
  type        = string
}

variable "bandwidth_size" {
  description = "带宽大小（Mbit/s）"
  type        = number
}

resource "huaweicloud_vpc_eip" "test" {
  publicip {
    type = "5_bgp"
  }

  bandwidth {
    name        = var.bandwidth_name
    size        = var.bandwidth_size
    share_type  = "PER"
    charge_mode = "traffic"
  }
}
```

**参数说明**：
- **publicip.type**：公网IP类型
- **bandwidth.name**：带宽名称
- **bandwidth.size**：带宽大小
- **bandwidth.share_type**：带宽共享类型
- **bandwidth.charge_mode**：计费模式

#### 6. WAF专业版实例（huaweicloud_waf_dedicated_instance）

在默认region（默认继承当前provider块中所指定的region）下创建WAF专业版实例，用于提供Web应用防护服务。

```hcl
variable "waf_instance_name" {
  description = "WAF实例名称"
  type        = string
}

resource "huaweicloud_waf_dedicated_instance" "test" {
  name               = var.waf_instance_name
  available_zone     = data.huaweicloud_availability_zones.test.names[0]
  specification_code = "waf.instance.professional"
  
  vpc_id            = huaweicloud_vpc.test.id
  subnet_id         = huaweicloud_vpc_subnet.test.id
  security_group_id = huaweicloud_networking_secgroup.test.id
  
  enterprise_project_id = "0"
}
```

**参数说明**：
- **name**：WAF实例名称
- **available_zone**：可用区
- **specification_code**：规格编码
- **vpc_id**：VPC ID
- **subnet_id**：子网ID
- **security_group_id**：安全组ID
- **enterprise_project_id**：企业项目ID

### 可扩展配置

#### 1. HTTPS安全组规则（huaweicloud_networking_secgroup_rule）

在默认region（默认继承当前provider块中所指定的region）下创建允许HTTPS访问的安全组规则。

```hcl
resource "huaweicloud_networking_secgroup_rule" "allow_https" {
  security_group_id = huaweicloud_networking_secgroup.test.id
  direction         = "ingress"
  ethertype        = "IPv4"
  protocol         = "tcp"
  port_range_min   = 443
  port_range_max   = 443
  remote_ip_prefix = "0.0.0.0/0"
  description      = "允许HTTPS访问"
}
```

**参数说明**：
- **security_group_id**：安全组ID
- **direction**：规则方向
- **ethertype**：网络协议版本
- **protocol**：协议类型
- **port_range_min**：起始端口
- **port_range_max**：结束端口
- **remote_ip_prefix**：允许访问的IP范围
- **description**：规则描述

> 该规则用于允许HTTPS（443）端口访问。建议根据实际业务需求配置remote_ip_prefix，遵循最小权限原则。

## 部署流程

1. 创建VPC和子网
2. 配置安全组和规则
3. 申请弹性公网IP
4. 创建WAF专业版实例
5. 配置WAF实例参数

## 操作步骤

1. **准备工作**
   - 安装Terraform
   - 配置华为云认证信息
   - 创建工作目录

2. **创建Terraform配置文件**
   ```bash
   touch main.tf
   touch variables.tf
   ```

3. **初始化和部署**
   ```bash
   terraform init
   terraform plan
   terraform apply
   ```

4. **验证部署**
   - 登录WAF控制台
   - 检查实例状态
   - 配置防护域名
   - 验证防护效果

## 注意事项

1. **网络规划**
   - 合理规划VPC网段
   - 配置必要的安全组规则
   - 确保网络连通性

2. **安全配置**
   - 及时更新WAF规则库
   - 定期检查安全日志
   - 配置告警通知

3. **成本控制**
   - 选择合适的实例规格
   - 合理配置带宽大小
   - 关注资源使用情况

## 最佳实践效果

通过本最佳实践的实施，您将获得：

1. 自动化部署的WAF专业版实例
2. 独享的Web安全防护能力
3. 可定制的安全防护策略
4. 合规的安全防护体系

## 参考信息

- [WAF产品文档](https://support.huaweicloud.com/waf/index.html)
- [Terraform华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [WAF最佳实践](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/waf/waf-dedicated-instance) 