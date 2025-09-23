# Preparation Before Deploying Huawei Cloud Resources

Before starting to use Terraform, you need to complete the following preparation steps.

## Install Terraform

Terraform installation methods vary by operating system. We provide detailed installation guide documentation: [How to Install Terraform](how_to_install_terraform.md)
Please follow the instructions in the documentation.

## Prepare Working Directory for Deployment Scripts

Create a new working directory and prepare Terraform configuration files (with .tf extension, such as main.tf):

```bash
mkdir -p terraform-demo
cd terraform-demo
touch main.tf
```

In `main.tf`, you will define the HCL scripts needed to deploy Huawei Cloud resources.

## Configure Huawei Cloud Authentication Information

### Method 1: Environment Variable Configuration (Recommended)

You can configure Huawei Cloud authentication information by setting the following environment variables:

```bash
# Required Configuration
export HW_ACCESS_KEY="your_ak"     # AK, generated in console "My Credentials - Access Keys"
export HW_SECRET_KEY="your_sk"     # SK, generated in console "My Credentials - Access Keys"
export HW_REGION_NAME="cn-north-4" # Region code, e.g.: cn-north-4

# Optional Configuration
export HW_ENTERPRISE_PROJECT_ID="your_eps_id" # Enterprise Project ID (if needed)
export HW_DOMAIN_NAME="your_domain_name"      # Account name (if cross-account operations needed)
```

### Method 2: Provider Configuration Block

Configure authentication information directly in Terraform configuration files:

```hcl
terraform {
  required_providers {
    huaweicloud = {
      source  = "huaweicloud/huaweicloud"
      version = ">= 1.xx.x" # Load the latest provider version not lower than 1.xx.x, this configuration is still valid for local providers
    }
  }
}

provider "huaweicloud" {
  access_key = "your_ak"    # Required, AK, generated in console "My Credentials - Access Keys"
  secret_key = "your_sk"    # Required, SK, generated in console "My Credentials - Access Keys"
  region     = "cn-north-4" # Required, region code, e.g.: cn-north-4

  # The following are optional configurations
  enterprise_project_id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" # Enterprise Project ID, if the account has enterprise management enabled, this ID defaults to "0"
  domain_name           = "your_domain_name"                     # Account name
}
```

## Reference Information

- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Terraform Official Documentation](https://www.terraform.io/docs/index.html)
