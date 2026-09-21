# 部署自定义模板扫描规则

## 应用场景

数据安全中心（Data Security Center，DSC）提供敏感数据识别、数据脱敏、数据水印和数据安全审计等能力，帮助企业构建可视、可控、可审计的数据安全防护体系。在敏感数据识别过程中，除了使用内置的识别规则外，用户往往需要根据自身业务特征自定义扫描模板与扫描规则，以便更精准地识别特定格式的敏感数据。

本最佳实践将介绍如何使用Terraform自动化部署自定义模板扫描规则，依次创建自定义扫描安全级别、扫描模板、扫描模板分类以及关联模板的正则表达式扫描规则，帮助您快速构建符合业务需求的敏感数据识别能力。

## 相关资源/数据源

本最佳实践涉及以下主要资源：

### 资源

- [DSC扫描安全级别（huaweicloud_dsc_scan_security_level）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dsc_scan_security_level)
- [DSC扫描模板（huaweicloud_dsc_scan_template）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dsc_scan_template)
- [DSC扫描模板分类（huaweicloud_dsc_scan_template_classification）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dsc_scan_template_classification)
- [DSC扫描规则（huaweicloud_dsc_scan_rule）](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dsc_scan_rule)

### 资源/数据源依赖关系

```
huaweicloud_dsc_scan_security_level
    └── huaweicloud_dsc_scan_rule

huaweicloud_dsc_scan_template
    ├── huaweicloud_dsc_scan_template_classification
    │       └── huaweicloud_dsc_scan_rule
    └── huaweicloud_dsc_scan_rule
```

## 操作步骤

### 1. 脚本准备

在指定工作空间中准备好用于编写当前最佳实践脚本的TF文件（如main.tf），确保其中（也可以是其他同级目录下的TF文件）包含部署资源所需的provider版本声明和华为云鉴权信息。
配置介绍参考[部署华为云资源前的准备工作](../../introductions/prepare_before_deploy.md)一文中的介绍。

### 2. 创建扫描安全级别

在TF文件（如main.tf）中添加以下脚本以创建DSC扫描安全级别：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DSC扫描安全级别资源
variable "security_level_name" {
  description = "The name of the security level"
  type        = string
}

variable "security_level_color_number" {
  description = "The color number of the security level displayed on the console"
  type        = number
  default     = 6
}

variable "security_level_description" {
  description = "The description of the security level"
  type        = string
  default     = ""
}

resource "huaweicloud_dsc_scan_security_level" "test" {
  security_level_name = var.security_level_name
  color_number        = var.security_level_color_number
  security_level_desc = var.security_level_description
}
```

**参数说明**：
- **security_level_name**：安全级别名称，通过引用输入变量 security_level_name 进行赋值
- **color_number**：安全级别在控制台显示的颜色编号，通过引用输入变量 security_level_color_number 进行赋值，默认值为6
- **security_level_desc**：安全级别描述，通过引用输入变量 security_level_description 进行赋值

### 3. 创建扫描模板

在TF文件（如main.tf）中添加以下脚本以创建DSC扫描模板：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DSC扫描模板资源
variable "scan_template_name" {
  description = "The name of the scan template"
  type        = string
}

variable "scan_template_description" {
  description = "The description of the scan template"
  type        = string
  default     = "Created_by_terraform_script"
}

resource "huaweicloud_dsc_scan_template" "test" {
  action             = "ADD"
  name               = var.scan_template_name
  description        = var.scan_template_description
  add_built_in_rules = false
}
```

**参数说明**：
- **action**：模板操作类型，固定为`ADD`，表示新增扫描模板
- **name**：扫描模板名称，通过引用输入变量 scan_template_name 进行赋值
- **description**：扫描模板描述，通过引用输入变量 scan_template_description 进行赋值
- **add_built_in_rules**：是否添加内置规则，设置为`false`表示不添加内置规则，仅使用本实践创建的自定义扫描规则

### 4. 创建扫描模板分类

在TF文件（如main.tf）中添加以下脚本以创建DSC扫描模板分类：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DSC扫描模板分类资源
variable "classification_name" {
  description = "The name of the scan template classification"
  type        = string
}

resource "huaweicloud_dsc_scan_template_classification" "test" {
  template_id         = huaweicloud_dsc_scan_template.test.id
  classification_name = var.classification_name
}
```

**参数说明**：
- **template_id**：所属扫描模板ID，引用上一步创建的扫描模板的ID进行赋值
- **classification_name**：分类名称，通过引用输入变量 classification_name 进行赋值

### 5. 创建扫描规则

在TF文件（如main.tf）中添加以下脚本以创建DSC扫描规则，并将其关联到扫描模板、分类和安全级别：

```hcl
# 在指定region（region参数缺省时默认继承当前provider块中所指定的region）下创建DSC扫描规则资源
variable "scan_rule_name" {
  description = "The name of the scan rule"
  type        = string
}

variable "scan_rule_match_rate" {
  description = "The match rate of the scan rule"
  type        = number
  default     = 1
}

variable "scan_rule_description" {
  description = "The description of the scan rule"
  type        = string
  default     = ""
}

resource "huaweicloud_dsc_scan_rule" "test" {
  rule_name      = var.scan_rule_name
  rule_type      = "REGEX"
  category       = "BUILT_SELF"
  logic_operator = "AND"
  match_rate     = var.scan_rule_match_rate
  min_match      = 1
  rule_desc      = var.scan_rule_description

  content {
    effective_mode = "NOT_IN"
    location       = "NAME"
    rule_content   = "bphone"
  }

  content {
    effective_mode = "IN"
    location       = "REMARK"
    rule_content   = "telephone number"
  }

  templates {
    template_id       = huaweicloud_dsc_scan_template.test.id
    classification_id = huaweicloud_dsc_scan_template_classification.test.id
    security_level_id = huaweicloud_dsc_scan_security_level.test.id
    is_used           = "true"
  }
}
```

**参数说明**：
- **rule_name**：扫描规则名称，通过引用输入变量 scan_rule_name 进行赋值
- **rule_type**：规则类型，固定为`REGEX`，表示正则表达式规则
- **category**：规则类别，固定为`BUILT_SELF`，表示自定义规则
- **logic_operator**：多个匹配内容之间的逻辑关系，固定为`AND`
- **match_rate**：规则匹配率，通过引用输入变量 scan_rule_match_rate 进行赋值
- **min_match**：最小匹配次数，固定为1
- **rule_desc**：规则描述，通过引用输入变量 scan_rule_description 进行赋值
- **content**：规则匹配内容块，可定义多个，其中 effective_mode 表示生效模式（`IN`/`NOT_IN`），location 表示匹配位置（如`NAME`、`REMARK`），rule_content 表示匹配内容
- **templates**：规则关联的模板信息块，其中 template_id、classification_id、security_level_id 分别引用前面创建的扫描模板、扫描模板分类和扫描安全级别的ID，is_used 表示是否启用该规则

### 6. 预设资源部署所需的入参（可选）

本实践中，部分资源使用了输入变量对配置内容进行赋值，这些输入参数在后续部署时需要手工输入。
同时，Terraform提供了通过`tfvars`文件预设这些配置的方法，可以避免每次执行时重复输入。

在工作目录下创建`terraform.tfvars`文件，示例内容如下：

```hcl
# 认证变量
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# 资源变量
security_level_name = "tfscanlevel"
scan_template_name  = "tfscantemplate"
classification_name = "tfclassification"
scan_rule_name      = "tfscanrule"
```

**使用方法**：

1. 将上述内容保存为工作目录下的`terraform.tfvars`文件（该文件名可使用户在执行terraform命令时自动导入该`tfvars`文件中的内容，其他命名则需要在tfvars前补充`.auto`定义，如`variables.auto.tfvars`）
2. 根据实际需要修改参数值
3. 执行`terraform plan`或`terraform apply`时，Terraform会自动读取该文件中的变量值

除了使用`terraform.tfvars`文件外，还可以通过以下方式设置变量值：

1. 命令行参数：`terraform apply -var="security_level_name=my-level"`
2. 环境变量：`export TF_VAR_security_level_name=my-level`
3. 自定义命名的变量文件：`terraform apply -var-file="custom.tfvars"`

> 注意：如果同一个变量通过多种方式进行设置，Terraform会按照以下优先级使用变量值：命令行参数 > 变量文件 > 环境变量 > 默认值。

### 7. 初始化并应用Terraform配置

完成以上脚本配置后，执行以下步骤来创建资源：

1. 运行 `terraform init` 初始化环境
2. 运行 `terraform plan` 查看资源创建计划
3. 确认资源计划无误后，运行 `terraform apply` 开始创建自定义模板扫描规则
4. 运行 `terraform show` 查看已创建的自定义模板扫描规则

## 参考信息

- [华为云数据安全中心产品文档](https://support.huaweicloud.com/dsc/index.html)
- [华为云Provider文档](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DSC自定义模板扫描规则最佳实践源码参考](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dsc/scan-rule-with-custom-template)
