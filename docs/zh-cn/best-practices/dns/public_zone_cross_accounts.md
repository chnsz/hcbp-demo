# 跨账号创建公网域名

## 应用场景

云解析服务（Domain Name Service, DNS）是华为云提供的高可用、高性能的域名解析服务，支持公网域名解析和私网域名解析。DNS服务提供智能解析、负载均衡、健康检查等功能，帮助用户实现域名的智能调度和故障转移。

跨账号创建公网域名是DNS服务中的高级功能，允许一个账号（主账号）授权另一个账号（目标账号）在主账号的域名下创建和管理子域名。该功能在多账号场景中特别有用，例如当组织需要将子域名管理委托给不同的部门或团队，同时保持对主域名的集中控制。通过跨账号授权，企业可以实现分层域名管理，提高运营效率，增强不同账号之间的安全隔离。本最佳实践将介绍如何使用Terraform自动化部署跨账号公网域名创建，包括域名授权、记录集创建、授权验证和子域名创建。

## 相关资源/数据源

本最佳实践涉及以下主要资源和数据源：

### 数据源

- [DNS域名查询数据源（data.huaweicloud_dns_zones）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dns_zones)

### 资源

- [DNS域名授权资源（huaweicloud_dns_zone_authorization）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dns_zone_authorization)
- [DNS记录集资源（huaweicloud_dns_recordset）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dns_recordset)
- [DNS域名授权验证资源（huaweicloud_dns_zone_authorization_verify）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dns_zone_authorization_verify)
- [DNS公网域名资源（huaweicloud_dns_zone）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dns_zone)

### 资源/数据源依赖关系

```
data.huaweicloud_dns_zones.test
    └── huaweicloud_dns_zone_authorization.test

huaweicloud_dns_zone_authorization.test
    ├── huaweicloud_dns_recordset.test
    └── huaweicloud_dns_zone_authorization_verify.test

huaweicloud_dns_recordset.test
    └── huaweicloud_dns_zone_authorization_verify.test

huaweicloud_dns_zone_authorization_verify.test
    └── huaweicloud_dns_zone.test
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

> 注意：本最佳实践涉及跨账号操作，需要配置两个provider：一个用于主账号（domain_master），一个用于目标账号。主账号provider用于查询主域名和创建记录集，目标账号provider用于创建子域名。

### 2. 配置跨账号操作的Provider

在TF文件（如main.tf）中添加以下脚本以配置主账号和目标账号的provider：

```hcl
# 配置主账号（域名所有者）的provider
provider "huaweicloud" {
  alias      = "domain_master"
  region     = var.region_name
  access_key = var.access_key
  secret_key = var.secret_key
}

# 配置目标账号（子域名创建者）的provider
provider "huaweicloud" {
  alias      = "domain_target"
  region     = var.region_name
  access_key = var.target_account_access_key
  secret_key = var.target_account_secret_key
}
```

**参数说明**：
- **alias**：Provider别名，用于区分主账号和目标账号的provider
- **region**：资源所在的区域，通过引用输入变量 `region_name` 进行赋值
- **access_key**：认证使用的访问密钥，主账号使用 `var.access_key`，目标账号使用 `var.target_account_access_key`
- **secret_key**：认证使用的秘密密钥，主账号使用 `var.secret_key`，目标账号使用 `var.target_account_secret_key`

> 注意：主账号provider用于查询主域名和创建用于授权验证的记录集。目标账号provider用于在授权验证通过后创建子域名。

### 3. 通过数据源查询DNS域名信息

在TF文件（如main.tf）中添加以下脚本以告知Terraform进行一次数据源查询，其查询结果用于创建DNS域名授权资源：

```hcl
variable "main_domain_name" {
  description = "主域名的名称"
  type        = string
}

# 获取指定region（region参数缺省时默认继承当前provider块中所指定的region）下符合条件的DNS域名信息，用于创建DNS域名授权资源
data "huaweicloud_dns_zones" "test" {
  provider = huaweicloud.domain_master

  name        = var.main_domain_name
  zone_type   = "public"
  search_mode = "equal"
}
```

**参数说明**：
- **provider**：Provider别名，指定为 `huaweicloud.domain_master` 以使用主账号provider
- **name**：DNS域名的名称，通过引用输入变量 `main_domain_name` 进行赋值
- **zone_type**：DNS域名的类型，设置为"public"表示查询公网域名
- **search_mode**：搜索模式，设置为"equal"表示执行精确匹配搜索

### 4. 创建DNS域名授权资源

在TF文件中添加以下脚本以告知Terraform创建DNS域名授权资源：

```hcl
variable "sub_domain_prefix" {
  description = "子域名的前缀"
  type        = string
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DNS域名授权资源
resource "huaweicloud_dns_zone_authorization" "test" {
  depends_on = [data.huaweicloud_dns_zones.test]

  zone_name = format("%s.%s", var.sub_domain_prefix, try(data.huaweicloud_dns_zones.test.zones[0].name, "master_domain_not_found"))
}
```

**参数说明**：
- **depends_on**：显式依赖声明，确保数据源查询完成后再创建授权资源
- **zone_name**：要授权的域名名称，使用 `format` 函数将子域名前缀与主域名名称组合而成

> 注意：域名授权资源在目标账号上下文中创建，允许目标账号管理主域名下的子域名。

### 5. 创建DNS记录集资源

在TF文件中添加以下脚本以告知Terraform创建DNS记录集资源（用于授权验证）：

```hcl
variable "recordset_type" {
  description = "记录集的类型"
  type        = string
  default     = "TXT"
}

variable "recordset_ttl" {
  description = "记录集的生存时间（TTL）"
  type        = number
  default     = 300
}

# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DNS记录集资源，用于授权验证
resource "huaweicloud_dns_recordset" "test" {
  provider = huaweicloud.domain_master

  zone_id = try(data.huaweicloud_dns_zones.test.zones[0].id, null)
  name    = format("%s.%s", try(huaweicloud_dns_zone_authorization.test.record[0].host, "host_not_found"), try(data.huaweicloud_dns_zones.test.zones[0].name, "master_domain_not_found"))
  type    = var.recordset_type
  ttl     = var.recordset_ttl
  records = ["\"${huaweicloud_dns_zone_authorization.test.record[0].value}\""]

  provisioner "local-exec" {
    command = "sleep 10"
  }
}
```

**参数说明**：
- **provider**：Provider别名，指定为 `huaweicloud.domain_master` 以使用主账号provider
- **zone_id**：DNS域名的ID，从查询到的DNS域名数据源中获取
- **name**：记录集的名称，使用 `format` 函数将授权资源中的host与主域名名称组合而成
- **type**：记录集的类型，通过引用输入变量 `recordset_type` 进行赋值，默认为"TXT"
- **ttl**：记录集的生存时间，通过引用输入变量 `recordset_ttl` 进行赋值，默认为300秒
- **records**：记录值，从授权资源中获取，TXT记录需要用引号包裹
- **provisioner**：local-exec provisioner，在创建记录集后等待10秒，确保DNS传播完成

> 注意：记录集在主账号中创建，用于验证授权。provisioner确保在进入验证步骤之前有足够的时间进行DNS传播。

### 6. 创建DNS域名授权验证资源

在TF文件中添加以下脚本以告知Terraform创建DNS域名授权验证资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DNS域名授权验证资源，用于验证授权
resource "huaweicloud_dns_zone_authorization_verify" "test" {
  depends_on = [huaweicloud_dns_recordset.test]

  authorization_id = huaweicloud_dns_zone_authorization.test.id
}
```

**参数说明**：
- **depends_on**：显式依赖声明，确保记录集创建完成后再验证授权
- **authorization_id**：DNS域名授权资源的ID，引用前面创建的授权资源

> 注意：授权验证会检查在主账号中创建的TXT记录是否与预期值匹配。只有验证成功后，目标账号才能创建子域名。

### 7. 创建DNS公网域名资源

在TF文件中添加以下脚本以告知Terraform创建DNS公网域名资源：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DNS公网域名资源
resource "huaweicloud_dns_zone" "test" {
  depends_on = [huaweicloud_dns_zone_authorization_verify.test]

  name      = format("%s.%s", var.sub_domain_prefix, try(data.huaweicloud_dns_zones.test.zones[0].name, "master_domain_not_found"))
  zone_type = "public"
}
```

**参数说明**：
- **depends_on**：显式依赖声明，确保授权验证通过后再创建域名
- **name**：DNS域名的名称，使用 `format` 函数将子域名前缀与主域名名称组合而成
- **zone_type**：DNS域名的类型，设置为"public"表示创建公网域名

> 注意：DNS域名在授权验证成功后，在目标账号上下文中创建。域名名称必须与授权的子域名名称匹配。

### 8. 预设资源部署所需的入参（可选）

本实践中，部分资源、数据源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 目标账号鉴权配置
target_account_access_key = "access_key_of_target_account"
target_account_secret_key = "secret_key_of_target_account"

# DNS域名配置
main_domain_name  = "domain_name_of_target_account"
sub_domain_prefix = "dev"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="main_domain_name=example.com" -var="sub_domain_prefix=dev"`
2. 环境变量：`export TF_VAR_main_domain_name=example.com`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 9. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建DNS域名授权、记录集、授权验证和公网域名
4. 运行 `terraform show` 查看已创建的DNS资源详情

## 参考信息

- [华为云DNS产品文档](https://support.huaweicloud.com/dns/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DNS跨账号创建公网域名最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dns/public-zone-cross-accounts)
