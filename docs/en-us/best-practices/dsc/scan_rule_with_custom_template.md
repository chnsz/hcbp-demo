# Deploy Custom Scan Rule with Template

## Application Scenario

Data Security Center (DSC) provides capabilities such as sensitive data identification, data masking, data watermarking, and data security auditing, helping enterprises build a visible, controllable, and auditable data security protection system. During sensitive data identification, in addition to built-in identification rules, users often need to customize scan templates and scan rules based on their own business characteristics to identify sensitive data in specific formats more accurately.

This best practice will introduce how to use Terraform to automatically deploy a custom scan rule with a template, creating a custom scan security level, a scan template, a scan template classification, and a regular expression scan rule associated with the template, helping you quickly build sensitive data identification capabilities that meet business requirements.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [DSC Scan Security Level (huaweicloud_dsc_scan_security_level)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dsc_scan_security_level)
- [DSC Scan Template (huaweicloud_dsc_scan_template)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dsc_scan_template)
- [DSC Scan Template Classification (huaweicloud_dsc_scan_template_classification)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dsc_scan_template_classification)
- [DSC Scan Rule (huaweicloud_dsc_scan_rule)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dsc_scan_rule)

### Resource/Data Source Dependencies

```
huaweicloud_dsc_scan_security_level
    └── huaweicloud_dsc_scan_rule

huaweicloud_dsc_scan_template
    ├── huaweicloud_dsc_scan_template_classification
    │       └── huaweicloud_dsc_scan_rule
    └── huaweicloud_dsc_scan_rule
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a Scan Security Level

Add the following script in the TF file (such as main.tf) to create a DSC scan security level:

```hcl
# Create a DSC scan security level resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

**Parameter Description**:
- **security_level_name**: The name of the security level, assigned by referencing the input variable security_level_name
- **color_number**: The color number of the security level displayed on the console, assigned by referencing the input variable security_level_color_number, defaulting to 6
- **security_level_desc**: The description of the security level, assigned by referencing the input variable security_level_description

### 3. Create a Scan Template

Add the following script in the TF file (such as main.tf) to create a DSC scan template:

```hcl
# Create a DSC scan template resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

**Parameter Description**:
- **action**: The template operation type, fixed to `ADD`, indicating that a new scan template is created
- **name**: The name of the scan template, assigned by referencing the input variable scan_template_name
- **description**: The description of the scan template, assigned by referencing the input variable scan_template_description
- **add_built_in_rules**: Whether to add built-in rules, set to `false` so that only the custom scan rule created in this best practice is used

### 4. Create a Scan Template Classification

Add the following script in the TF file (such as main.tf) to create a DSC scan template classification:

```hcl
# Create a DSC scan template classification resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "classification_name" {
  description = "The name of the scan template classification"
  type        = string
}

resource "huaweicloud_dsc_scan_template_classification" "test" {
  template_id         = huaweicloud_dsc_scan_template.test.id
  classification_name = var.classification_name
}
```

**Parameter Description**:
- **template_id**: The ID of the scan template to which the classification belongs, assigned by referencing the ID of the scan template created in the previous step
- **classification_name**: The name of the classification, assigned by referencing the input variable classification_name

### 5. Create a Scan Rule

Add the following script in the TF file (such as main.tf) to create a DSC scan rule and associate it with the scan template, classification, and security level:

```hcl
# Create a DSC scan rule resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
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

**Parameter Description**:
- **rule_name**: The name of the scan rule, assigned by referencing the input variable scan_rule_name
- **rule_type**: The rule type, fixed to `REGEX`, indicating a regular expression rule
- **category**: The rule category, fixed to `BUILT_SELF`, indicating a custom rule
- **logic_operator**: The logical relationship between multiple match contents, fixed to `AND`
- **match_rate**: The match rate of the scan rule, assigned by referencing the input variable scan_rule_match_rate
- **min_match**: The minimum number of matches, fixed to 1
- **rule_desc**: The description of the scan rule, assigned by referencing the input variable scan_rule_description
- **content**: The rule match content block, which can be defined multiple times, where effective_mode indicates the effective mode (`IN`/`NOT_IN`), location indicates the match location (such as `NAME` and `REMARK`), and rule_content indicates the match content
- **templates**: The template information block associated with the rule, where template_id, classification_id, and security_level_id reference the IDs of the scan template, scan template classification, and scan security level created above respectively, and is_used indicates whether the rule is enabled

### 6. Preset Input Parameters Required for Resource Deployment (Optional)

In this best practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# Resource variables
security_level_name = "tfscanlevel"
scan_template_name  = "tfscantemplate"
classification_name = "tfclassification"
scan_rule_name      = "tfscanrule"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="security_level_name=my-level"`
2. Environment variables: `export TF_VAR_security_level_name=my-level`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 7. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the custom scan rule with a template
4. Run `terraform show` to view the created custom scan rule with a template

## Reference Information

- [Huawei Cloud Data Security Center Product Documentation](https://support.huaweicloud.com/dsc/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DSC Custom Scan Rule with Template](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dsc/scan-rule-with-custom-template)
