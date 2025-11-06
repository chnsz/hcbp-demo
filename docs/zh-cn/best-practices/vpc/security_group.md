# 部署安全组

## 应用场景

安全组（Security Group）是华为云VPC中用于控制网络访问的虚拟防火墙，通过配置安全组规则，可以精确控制云服务器、数据库等资源的网络访问权限。安全组支持入方向和出方向规则配置，能够有效保护云上资源的安全。本最佳实践将介绍如何使用Terraform自动化部署安全组及其规则配置。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [安全组资源（huaweicloud_networking_secgroup）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup)
- [安全组规则资源（huaweicloud_networking_secgroup_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/networking_secgroup_rule)

### 资源/数据源依赖关系

```
huaweicloud_networking_secgroup
    └── huaweicloud_networking_secgroup_rule
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建安全组资源

在TF文件（如main.tf）中添加以下脚本以告知Terraform创建安全组资源：

```hcl
variable "security_group_name" {
  description = "The name of the security group."
  type        = string
}

# 创建安全组资源
resource "huaweicloud_networking_secgroup" "test" {
  name                 = var.security_group_name
  delete_default_rules = true
}
```

**参数说明**：
- **name**：安全组名称，通过引用输入变量security_group_name进行赋值
- **delete_default_rules**：是否删除默认规则，设置为true表示删除默认规则

### 3. 创建安全组规则资源

在TF文件中添加以下脚本以告知Terraform批量创建安全组规则：

```hcl
variable "security_group_rule_configurations" {
  description = "The list of security group rule configurations. Each item is a map with keys: direction, ethertype, protocol, ports, remote_ip_prefix."
  type = list(object({
    direction        = optional(string, "ingress")
    ethertype        = optional(string, "IPv4")
    protocol         = optional(string, null)
    ports            = optional(string, null)
    remote_ip_prefix = optional(string, "0.0.0.0/0")
  }))
  nullable = false
}

# 创建安全组规则资源
resource "huaweicloud_networking_secgroup_rule" "test" {
  count = length(var.security_group_rule_configurations)

  direction         = lookup(var.security_group_rule_configurations[count.index], "direction", "ingress")
  ethertype         = lookup(var.security_group_rule_configurations[count.index], "ethertype", "IPv4")
  protocol          = lookup(var.security_group_rule_configurations[count.index], "protocol", null)
  ports             = lookup(var.security_group_rule_configurations[count.index], "ports", null)
  remote_ip_prefix  = lookup(var.security_group_rule_configurations[count.index], "remote_ip_prefix", "0.0.0.0/0")
  security_group_id = huaweicloud_networking_secgroup.test.id
}
```

**参数说明**：
- **direction**：规则方向，通过引用输入变量security_group_rule_configurations进行赋值，默认值为"ingress"
- **ethertype**：以太网类型，通过引用输入变量security_group_rule_configurations进行赋值，默认值为"IPv4"
- **protocol**：协议类型，通过引用输入变量security_group_rule_configurations进行赋值，默认值为null
- **ports**：端口范围，通过引用输入变量security_group_rule_configurations进行赋值，默认值为null
- **remote_ip_prefix**：远端IP地址段，通过引用输入变量security_group_rule_configurations进行赋值，默认值为"0.0.0.0/0"
- **security_group_id**：安全组ID，引用前面创建的安全组资源的ID

### 4. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`.tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 华为云认证信息
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# 安全组配置
security_group_name = "tf_test_security_group"
security_group_rule_configurations = [
  # Allow all IPv4 ingress traffic of the ICMP protocol
  {
    direction        = "ingress"
    ethertype        = "IPv4"
    protocol         = "icmp"
    ports            = null
    remote_ip_prefix = "0.0.0.0/0"
  },
  # Allow some ports for IPv4 ingress traffic of the TCP protocol
  {
    direction        = "ingress"
    ethertype        = "IPv4"
    protocol         = "tcp"
    ports            = "22-23,443,3389,30100-30130"
    remote_ip_prefix = "0.0.0.0/0"
  },
  # Allow all IPv4 egress traffic
  {
    direction        = "egress"
    ethertype        = "IPv4"
    remote_ip_prefix = "0.0.0.0/0"
  },
  # Allow all IPv6 egress traffic
  {
    direction        = "egress"
    ethertype        = "IPv6"
    remote_ip_prefix = "::/0"
  }
]
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="security_group_name=my-security-group"`
2. 环境变量：`export TF_VAR_security_group_name=my-security-group`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 5. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建安全组及规则
4. 运行 `terraform show` 查看已创建的安全组及规则详情

## 参考信息

- [华为云VPC产品文档](https://support.huaweicloud.com/vpc/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [VPC安全组最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/vpc/security-group)
