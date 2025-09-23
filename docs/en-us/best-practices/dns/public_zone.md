# Deploy Public Zone

## Application Scenario

Domain Name Service (DNS) is a highly available, high-performance domain name resolution service provided by Huawei Cloud, supporting both public and private domain name resolution. DNS service provides intelligent resolution, load balancing, health check, and other functions, helping users achieve intelligent scheduling and failover of domain names.

Public zones are core functions in DNS service, used to manage internet-facing domain name resolution. Through public zones, enterprises can manage their website domains, API domains, mail server domains, etc., implementing domain name to IP address mapping. Public zones support multiple record types, including A records, AAAA records, CNAME records, MX records, etc., meeting resolution requirements for different application scenarios. This best practice will introduce how to use Terraform to automatically deploy DNS public zones, including domain creation, TTL configuration, DNSSEC settings, and router association.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [DNS Public Zone Resource (huaweicloud_dns_zone)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dns_zone)

### Resource/Data Source Dependencies

```
No dependencies
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create DNS Public Zone

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a DNS public zone resource:

```hcl
variable "dns_public_zone_name" {
  description = "Domain name"
  type        = string
}

variable "dns_public_zone_email" {
  description = "Administrator email address for managing the domain"
  type        = string
  default     = ""
}

variable "dns_public_zone_type" {
  description = "Domain type"
  type        = string
  default     = "public"
}

variable "dns_public_zone_description" {
  description = "Domain description"
  type        = string
}

variable "dns_public_zone_ttl" {
  description = "Time to live (TTL) of the domain"
  type        = number
  default     = 300
}

variable "dns_public_zone_enterprise_project_id" {
  description = "Enterprise project ID that the domain belongs to"
  type        = string
  default     = ""
}

variable "dns_public_zone_status" {
  description = "Domain status"
  type        = string
  default     = "ENABLE"
}

variable "dns_public_zone_dnssec" {
  description = "Whether to enable DNSSEC for the public domain"
  type        = string
  default     = "DISABLE"
}

variable "dns_public_zone_router" {
  description = "List of routers associated with the domain"
  type = list(object({
    router_id     = string
    router_region = string
  }))
  default = []
}

# Create a DNS public zone resource under the specified region (if the region parameter is omitted, it defaults to the region specified in the current provider block)
resource "huaweicloud_dns_zone" "test" {
  name                  = var.dns_public_zone_name
  email                 = var.dns_public_zone_email
  zone_type             = var.dns_public_zone_type
  description           = var.dns_public_zone_description
  ttl                   = var.dns_public_zone_ttl
  enterprise_project_id = var.dns_public_zone_enterprise_project_id
  status                = var.dns_public_zone_status
  dnssec                = var.dns_public_zone_dnssec

  dynamic "router" {
    for_each = var.dns_public_zone_router
    content {
      router_id     = router.value.router_id
      router_region = router.value.router_region
    }
  }
}
```

**Parameter Description**:
- **name**: Domain name, assigned by referencing the input variable dns_public_zone_name
- **email**: Administrator email address for managing the domain, assigned by referencing the input variable dns_public_zone_email
- **zone_type**: Domain type, assigned by referencing the input variable dns_public_zone_type, defaults to "public"
- **description**: Domain description, assigned by referencing the input variable dns_public_zone_description
- **ttl**: Time to live (TTL) of the domain, assigned by referencing the input variable dns_public_zone_ttl, defaults to 300 seconds
- **enterprise_project_id**: Enterprise project ID that the domain belongs to, assigned by referencing the input variable dns_public_zone_enterprise_project_id
- **status**: Domain status, assigned by referencing the input variable dns_public_zone_status, defaults to "ENABLE"
- **dnssec**: Whether to enable DNSSEC for the public domain, assigned by referencing the input variable dns_public_zone_dnssec, defaults to "DISABLE"
- **router.router_id**: Router ID, assigned by referencing the router_id field in the router list
- **router.router_region**: Router region, assigned by referencing the router_region field in the router list

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign values to configuration content. These input parameters need to be manually entered during subsequent deployments.
At the same time, Terraform provides a method to preset these configurations through `.tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Authentication information
region_name = "cn-north-4"
access_key  = "your-access-key"
secret_key  = "your-secret-key"

# DNS public zone configuration
dns_public_zone_name        = "tftest.yourname.com"
dns_public_zone_description = "tf_test_zone_desc"
dns_public_zone_ttl         = 3000
dns_public_zone_dnssec      = "ENABLE"
```

**Usage**:

1. Save the above content as `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using `terraform.tfvars` file, variable values can also be set in the following ways:

1. Command line parameters: `terraform apply -var="dns_public_zone_name=example.com" -var="dns_public_zone_description=My Domain"`
2. Environment variables: `export TF_VAR_dns_public_zone_name=example.com`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set in multiple ways, Terraform will use the variable value according to the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming the resource plan is correct, run `terraform apply` to start creating the DNS public zone
4. Run `terraform show` to view the details of the created DNS public zone

## Reference Information

- [Huawei Cloud DNS Product Documentation](https://support.huaweicloud.com/dns/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [DNS Best Practice Source Code Reference](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dns)
