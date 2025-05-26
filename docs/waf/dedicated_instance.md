# 使用Terraform部署WAF专业版实例

## 本最佳实践概述

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

### 涉及产品

- Web应用防火墙（WAF）：提供Web安全防护
- 虚拟私有云（VPC）：提供隔离的网络环境
- 弹性公网IP（EIP）：提供公网访问能力

## 资源/数据源设计

本最佳实践涉及以下主要资源：

1. **VPC网络（huaweicloud_vpc）**：
   - 用途：为WAF实例提供网络环境
   - 功能：提供安全隔离的网络空间
   - 特点：支持自定义网段配置
   - 关键配置：VPC网段、名称等
   - 安全特性：网络隔离
   - 输入：网段规划
   - 输出：VPC ID和网络环境

2. **VPC子网（huaweicloud_vpc_subnet）**：
   - 用途：在VPC中划分子网空间
   - 功能：管理IP地址分配
   - 特点：支持自定义网段和网关
   - 关键配置：子网网段、网关IP、DNS等
   - 安全特性：网络ACL控制
   - 输入：子网网段规划
   - 输出：子网ID和网络配置

3. **安全组（huaweicloud_networking_secgroup）**：
   - 用途：控制WAF实例的网络访问
   - 功能：定义入站和出站规则
   - 特点：支持精细化的访问控制
   - 关键配置：安全组规则、协议、端口等
   - 安全特性：基于规则的访问控制
   - 输入：访问控制需求
   - 输出：安全组ID和规则集

4. **安全组规则（huaweicloud_networking_secgroup_rule）**：
   - 用途：定义具体的访问控制规则
   - 功能：控制特定协议和端口的访问
   - 特点：支持入站和出站规则配置
   - 关键配置：协议类型、端口范围、源IP等
   - 安全特性：精细化的访问控制
   - 输入：协议、端口、IP范围
   - 输出：安全组规则配置

5. **弹性公网IP（huaweicloud_vpc_eip）**：
   - 用途：为WAF实例提供公网访问能力
   - 功能：提供固定的公网IP地址
   - 特点：支持带宽和计费方式配置
   - 关键配置：带宽大小、计费模式等
   - 安全特性：DDoS基础防护
   - 输入：带宽需求
   - 输出：EIP地址和ID

6. **WAF专业版实例（huaweicloud_waf_dedicated_instance）**：
   - 用途：提供Web应用防护服务
   - 功能：防御Web安全威胁
   - 特点：独享资源，性能保障
   - 关键配置：实例规格、网络配置等
   - 安全特性：Web安全防护
   - 输入：VPC配置、EIP配置
   - 输出：WAF实例ID和访问信息

### 资源依赖关系

```
VPC网络（huaweicloud_vpc）
    └── VPC子网（huaweicloud_vpc_subnet）
         └── WAF专业版实例（huaweicloud_waf_dedicated_instance）

安全组（huaweicloud_networking_secgroup）
    └── 安全组规则（huaweicloud_networking_secgroup_rule）
         └── WAF专业版实例（huaweicloud_waf_dedicated_instance）

弹性公网IP（huaweicloud_vpc_eip）
    └── WAF专业版实例（huaweicloud_waf_dedicated_instance）
```

## 详细配置

### 1. VPC配置（huaweicloud_vpc和huaweicloud_vpc_subnet）

**功能概述**

创建VPC网络环境和子网，为WAF专业版实例提供网络隔离。

**详细配置**

```hcl
resource "huaweicloud_vpc" "waf" {
  name = "waf-vpc"
  cidr = "172.16.0.0/16"

  tags = {
    environment = "production"
    owner       = "terraform"
  }
}

resource "huaweicloud_vpc_subnet" "waf" {
  name       = "waf-subnet"
  cidr       = "172.16.0.0/24"
  gateway_ip = "172.16.0.1"
  vpc_id     = huaweicloud_vpc.waf.id
  dns_list   = ["100.125.1.250", "100.125.21.250"] # 华为云DNS服务器

  tags = {
    environment = "production"
    owner       = "terraform"
  }
}
```

+ **VPC配置参数**：
  - **name**：VPC名称
  - **cidr**：VPC网段
  - **tags**：资源标签

+ **子网配置参数**：
  - **name**：子网名称
  - **cidr**：子网网段
  - **gateway_ip**：网关IP地址
  - **vpc_id**：所属VPC的ID
  - **dns_list**：DNS服务器列表
  - **tags**：资源标签

### 2. 安全组配置（huaweicloud_networking_secgroup和huaweicloud_networking_secgroup_rule）

**功能概述**

创建安全组和安全组规则，控制WAF专业版实例的网络访问。

**详细配置**

```hcl
resource "huaweicloud_networking_secgroup" "waf" {
  name        = "waf-secgroup"
  description = "Security group for WAF instance"
}

# 入站规则：允许HTTP访问
resource "huaweicloud_networking_secgroup_rule" "allow_http" {
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = 80
  port_range_max    = 80
  remote_ip_prefix  = "0.0.0.0/0"
  security_group_id = huaweicloud_networking_secgroup.waf.id
}

# 入站规则：允许HTTPS访问
resource "huaweicloud_networking_secgroup_rule" "allow_https" {
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = 443
  port_range_max    = 443
  remote_ip_prefix  = "0.0.0.0/0"
  security_group_id = huaweicloud_networking_secgroup.waf.id
}
```

+ **安全组配置参数**：
  - **name**：安全组名称
  - **description**：安全组描述

+ **安全组规则配置参数**：
  - **direction**：规则方向（入站/出站）
  - **ethertype**：网络协议版本
  - **protocol**：协议类型
  - **port_range_min**：起始端口
  - **port_range_max**：结束端口
  - **remote_ip_prefix**：允许访问的IP范围
  - **security_group_id**：安全组ID

### 3. 弹性公网IP配置（huaweicloud_vpc_eip）

**功能概述**

创建弹性公网IP，为WAF专业版实例提供公网访问能力。

**详细配置**

```hcl
resource "huaweicloud_vpc_eip" "waf" {
  publicip {
    type = "5_bgp"
  }

  bandwidth {
    name        = "waf-bandwidth"
    size        = 5
    share_type  = "PER"
    charge_mode = "traffic"
  }

  tags = {
    environment = "production"
    owner       = "terraform"
  }
}
```

+ **公网IP配置参数**：
  - **type**：公网IP类型，5_bgp表示动态BGP
  
+ **带宽配置参数**：
  - **name**：带宽名称
  - **size**：带宽大小（Mbit/s）
  - **share_type**：带宽类型，PER表示独享带宽
  - **charge_mode**：计费模式，traffic表示按流量计费

### 4. WAF专业版实例配置（huaweicloud_waf_dedicated_instance）

**功能概述**

创建WAF专业版实例，提供Web应用安全防护服务。

**详细配置**

```hcl
resource "huaweicloud_waf_dedicated_instance" "instance" {
  name               = "waf-instance"
  available_zone     = "cn-north-4a" # 根据实际区域配置
  specification_code = "waf.instance.professional" # 专业版规格

  vpc_id    = huaweicloud_vpc.waf.id
  subnet_id = huaweicloud_vpc_subnet.waf.id

  security_group {
    id = huaweicloud_networking_secgroup.waf.id
  }

  enterprise_project_id = "0" # 根据实际企业项目配置

  eip {
    eip_id = huaweicloud_vpc_eip.waf.id
  }
}
```

+ **基础配置参数**：
  - **name**：实例名称
  - **available_zone**：可用区
  - **specification_code**：规格代码

+ **网络配置参数**：
  - **vpc_id**：VPC ID
  - **subnet_id**：子网ID
  - **security_group**：安全组配置
  - **eip**：弹性公网IP配置

+ **企业项目参数**：
  - **enterprise_project_id**：企业项目ID

## 操作步骤

1. **准备工作**
   ```bash
   # 创建工作目录
   mkdir waf-terraform && cd waf-terraform
   
   # 创建main.tf文件
   touch main.tf
   
   # 创建variables.tf文件
   touch variables.tf
   ```

2. **初始化和部署**
   ```bash
   # 初始化Terraform
   terraform init
   
   # 检查配置
   terraform plan
   
   # 部署资源
   terraform apply
   ```

3. **验证部署**
   - 登录华为云控制台
   - 检查WAF实例状态
   - 验证网络连通性

## 最佳实践效果

通过本最佳实践的实施，您将获得：

1. 自动化部署的WAF专业版实例
2. 独享的Web安全防护能力
3. 可靠的网络环境
4. 灵活的配置管理

## 参考信息

- [华为云WAF产品文档](https://support.huaweicloud.com/waf/index.html)
- [Terraform华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [WAF最佳实践](https://support.huaweicloud.com/bestpractice-waf/waf_06_0001.html) 