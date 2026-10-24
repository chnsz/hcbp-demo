# Deploy Template with New Version

## Application Scenario

Resource Formation Service (RFS) is an Infrastructure as Code (IaC) service provided by Huawei Cloud, allowing users to describe the combination and dependencies of cloud resources through templates, and RFS orchestrates and delivers the resources in a unified way. In practice, templates often need to evolve with the business, so RFS supports publishing multiple versions for the same template, enabling iteration, traceability, and rollback of template content.

This best practice will introduce how to use Terraform to create an RFS template and publish a new version with an extended template body for the template. It first creates a template with an inline Terraform template body and its initial version, and then publishes a new version for the template based on the extended template body, demonstrating the complete process of template version management.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [RFS Template (huaweicloud_rfs_template)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rfs_template)
- [RFS Template Version (huaweicloud_rfs_template_version)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/rfs_template_version)

### Resource/Data Source Dependencies

```
huaweicloud_rfs_template
    └── huaweicloud_rfs_template_version
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, and ensure that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create an RFS Template

Add the following script to the TF file (such as main.tf) to create an RFS template:

```hcl
# Create an RFS template in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "template_name" {
  description = "The name of the RFS template"
  type        = string
}

variable "template_body" {
  description = "The Terraform template body of the RFS template"
  type        = string
}

variable "template_description" {
  description = "The description of the RFS template"
  type        = string
  default     = ""
}

variable "template_initial_version_description" {
  description = "The initial version description of the RFS template"
  type        = string
  default     = ""
}

resource "huaweicloud_rfs_template" "test" {
  template_name        = var.template_name
  template_body        = var.template_body
  template_description = var.template_description
  version_description  = var.template_initial_version_description
}
```

**Parameter Description**:
- **template_name**: The template name, assigned by referencing the input variable template_name. Only letters, digits, hyphens (-), and underscores (_) are allowed, and it must be unique within the region
- **template_body**: The template body, assigned by referencing the input variable template_body. It must be a valid Terraform configuration in HCL format; this example only stores the template definition and does not create the resources in the template body
- **template_description**: The template description, assigned by referencing the input variable template_description
- **version_description**: The initial version description of the template, assigned by referencing the input variable template_initial_version_description

### 3. Publish a New Version for the Template

Add the following script to the TF file (such as main.tf) to publish a new version for the template:

```hcl
# Publish a new version for the template in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "template_version_body" {
  description = "The Terraform template body of the RFS template version"
  type        = string
}

variable "template_version_description" {
  description = "The description of the RFS template version"
  type        = string
  default     = ""
}

resource "huaweicloud_rfs_template_version" "test" {
  template_name       = huaweicloud_rfs_template.test.template_name
  template_id         = huaweicloud_rfs_template.test.template_id
  template_body       = var.template_version_body
  version_description = var.template_version_description
}
```

**Parameter Description**:
- **template_name**: The template name, assigned by referencing the template_name of the template created in the previous step
- **template_id**: The template ID, assigned by referencing the template_id of the template created in the previous step
- **template_body**: The template body of the new version, assigned by referencing the input variable template_version_body. Resource definitions can be extended based on the initial template body
- **version_description**: The description of the new version, assigned by referencing the input variable template_version_description

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content, and these input parameters need to be manually entered during subsequent deployment.
Terraform also provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
template_name = "tf-test-template"
template_body = <<EOF
resource "huaweicloud_vpc" "test" {
  name = "tf-test-vpc"
  cidr = "172.16.0.0/16"
}
EOF

template_version_body = <<EOF
resource "huaweicloud_vpc" "test" {
  name = "tf-test-vpc"
  cidr = "172.16.0.0/16"
}

resource "huaweicloud_vpc_subnet" "test" {
  name       = "tf-test-subnet"
  vpc_id     = huaweicloud_vpc.test.id
  cidr       = "172.16.1.0/24"
  gateway_ip = "172.16.1.1"
}
EOF
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of the `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="template_name=my-template"`
2. Environment variables: `export TF_VAR_template_name=my-template`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the template and its new version
4. Run `terraform show` to view the created template and its new version

## Reference Information

- [Huawei Cloud Resource Formation Service (RFS) Product Documentation](https://support.huaweicloud.com/rfs/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For RFS Template with New Version](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/rfs/template-with-new-version)
