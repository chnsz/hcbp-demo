# Deploy Public Zone

## Application Scenario

Domain Name Service (DNS) is a highly available, high-performance domain name resolution service provided by Huawei Cloud, supporting both public and private domain name resolution. By creating a public zone, you can host your own domain name on Huawei Cloud DNS to achieve intelligent resolution, load balancing, and failover.

This best practice will introduce how to use Terraform to automatically deploy a DNS public zone, including zone creation, TTL configuration, DNSSEC settings, and router association.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Public Zone (huaweicloud_dns_zone)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dns_zone)

### Resource/Data Source Dependencies

```
huaweicloud_dns_zone
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For the configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a Public Zone

Add the following script in the TF file (such as main.tf) to create a public zone:

```hcl
# Create a public zone resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "dns_public_zone_name" {
  description = "The name of the zone"
  type        = string
}

variable "dns_public_zone_email" {
  description = "The email address of the administrator managing the zone"
  type        = string
  default     = ""
}

variable "dns_public_zone_type" {
  description = "The type of zone"
  type        = string
  default     = "public"
}

variable "dns_public_zone_description" {
  description = "The description of the zone"
  type        = string
}

variable "dns_public_zone_ttl" {
  description = "The time to live (TTL) of the zone"
  type        = number
  default     = 300
}

variable "dns_public_zone_enterprise_project_id" {
  description = "The enterprise project ID of the zone"
  type        = string
  default     = ""
}

variable "dns_public_zone_status" {
  description = "The status of the zone"
  type        = string
  default     = "ENABLE"
}

variable "dns_public_zone_dnssec" {
  description = "Whether to enable DNSSEC for a public zone"
  type        = string
  default     = "DISABLE"
}

variable "dns_public_zone_router" {
  description = "The list of the router of the zone"
  type        = list(object({
    router_id     = string
    router_region = string
  }))
  default     = []
}

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
- **name**: Assigned by referencing the input variable dns_public_zone_name, which is the name of the zone. Note that the name must end with a `.`
- **email**: Assigned by referencing the input variable dns_public_zone_email, which is the email address of the administrator managing the zone
- **zone_type**: Assigned by referencing the input variable dns_public_zone_type, which is the type of the zone, defaulting to `public`
- **description**: Assigned by referencing the input variable dns_public_zone_description, which is the description of the zone
- **ttl**: Assigned by referencing the input variable dns_public_zone_ttl, which is the time to live (TTL) of the zone, defaulting to 300
- **enterprise_project_id**: Assigned by referencing the input variable dns_public_zone_enterprise_project_id, which is the enterprise project ID of the zone
- **status**: Assigned by referencing the input variable dns_public_zone_status, which is the status of the zone, defaulting to `ENABLE`
- **dnssec**: Assigned by referencing the input variable dns_public_zone_dnssec, which sets whether to enable DNSSEC for the public zone, defaulting to `DISABLE`
- **router**: Assigned by referencing the input variable dns_public_zone_router, which is the list of routers (VPCs) associated with the zone, including `router_id` (the ID of the associated VPC) and `router_region` (the region of the VPC)

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Fill in based on the script variables; use placeholders for sensitive information
dns_public_zone_name        = "tftest.yourname.com"
dns_public_zone_description = "tf_test_zone_desc"
dns_public_zone_ttl         = 3000
dns_public_zone_dnssec      = "ENABLE"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of this `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="dns_public_zone_name=my-zone.com"`
2. Environment variables: `export TF_VAR_dns_public_zone_name=my-zone.com`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the public zone
4. Run `terraform show` to view the created public zone

## Reference Information

- [Huawei Cloud DNS Product Documentation](https://support.huaweicloud.com/dns/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DNS Public Zone](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dns/zone)
