# Deploy Public Zone Cross Accounts

## Application Scenario

Domain Name Service (DNS) is a highly available, high-performance domain name resolution service provided by Huawei Cloud, supporting both public and private domain name resolution. DNS service provides intelligent resolution, load balancing, health check, and other functions, helping users achieve intelligent scheduling and failover of domain names.

Cross-account public zone creation is an advanced feature in DNS service that allows one account (master account) to authorize another account (target account) to create and manage subdomains under the master account's domain. This feature is particularly useful in multi-account scenarios, such as when an organization needs to delegate subdomain management to different departments or teams while maintaining centralized control over the main domain. Through cross-account authorization, enterprises can implement hierarchical domain management, improve operational efficiency, and enhance security isolation between different accounts. This best practice will introduce how to use Terraform to automatically deploy cross-account public zone creation, including domain authorization, recordset creation, authorization verification, and subdomain zone creation.

## Related Resources/Data Sources

This best practice involves the following main resources and data sources:

### Data Sources

- [DNS Zones Query Data Source (data.huaweicloud_dns_zones)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/data-sources/dns_zones)

### Resources

- [DNS Zone Authorization Resource (huaweicloud_dns_zone_authorization)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dns_zone_authorization)
- [DNS Recordset Resource (huaweicloud_dns_recordset)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dns_recordset)
- [DNS Zone Authorization Verify Resource (huaweicloud_dns_zone_authorization_verify)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dns_zone_authorization_verify)
- [DNS Public Zone Resource (huaweicloud_dns_zone)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/dns_zone)

### Resource/Data Source Dependencies

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

## Implementation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the introduction in [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) for configuration introduction.

> Note: This best practice involves cross-account operations, requiring configuration of two providers: one for the master account (domain_master) and one for the target account. The master account provider is used to query the main domain and create recordsets, while the target account provider is used to create the subdomain zone.

### 2. Configure Providers for Cross-Account Operations

Add the following script to the TF file (such as main.tf) to configure providers for both the master account and target account:

```hcl
# Configure provider for master account (domain owner)
provider "huaweicloud" {
  alias     = "domain_master"
  region    = var.region_name
  access_key = var.access_key
  secret_key = var.secret_key
}

# Configure provider for target account (subdomain creator)
provider "huaweicloud" {
  alias     = "domain_target"
  region    = var.region_name
  access_key = var.target_account_access_key
  secret_key = var.target_account_secret_key
}
```

**Parameter Description**:
- **alias**: Provider alias, used to distinguish between master account and target account providers
- **region**: The region where resources are located, assigned by referencing the input variable `region_name`
- **access_key**: The access key for authentication, using `var.access_key` for master account and `var.target_account_access_key` for target account
- **secret_key**: The secret key for authentication, using `var.secret_key` for master account and `var.target_account_secret_key` for target account

> Note: The master account provider is used to query the main domain and create recordsets for authorization verification. The target account provider is used to create the subdomain zone after authorization is verified.

### 3. Query DNS Zones Information Through Data Source

Add the following script to the TF file (such as main.tf) to instruct Terraform to perform a data source query, the query results are used to create DNS zone authorization resources:

```hcl
variable "main_domain_name" {
  description = "The name of the main domain"
  type        = string
}

# Query DNS zones information that meets the conditions in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block), used to create DNS zone authorization resources
data "huaweicloud_dns_zones" "test" {
  provider = huaweicloud.domain_master

  name        = var.main_domain_name
  zone_type   = "public"
  search_mode = "equal"
}
```

**Parameter Description**:
- **provider**: The provider alias, specified as `huaweicloud.domain_master` to use the master account provider
- **name**: The name of the DNS zone, assigned by referencing the input variable `main_domain_name`
- **zone_type**: The type of the DNS zone, set to "public" to query public zones
- **search_mode**: The search mode, set to "equal" to perform exact match search

### 4. Create DNS Zone Authorization Resource

Add the following script to the TF file to instruct Terraform to create DNS zone authorization resources:

```hcl
variable "sub_domain_prefix" {
  description = "The prefix of the sub-domain"
  type        = string
}

# Create DNS zone authorization resource in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block)
resource "huaweicloud_dns_zone_authorization" "test" {
  depends_on = [data.huaweicloud_dns_zones.test]

  zone_name = format("%s.%s", var.sub_domain_prefix, try(data.huaweicloud_dns_zones.test.zones[0].name, "master_domain_not_found"))
}
```

**Parameter Description**:
- **depends_on**: Explicit dependency declaration, ensuring that the data source query completes before creating the authorization resource
- **zone_name**: The name of the zone to be authorized, constructed using the `format` function to combine the subdomain prefix with the main domain name

> Note: The zone authorization resource is created in the target account context, allowing the target account to manage the subdomain under the main domain.

### 5. Create DNS Recordset Resource

Add the following script to the TF file to instruct Terraform to create DNS recordset resources for authorization verification:

```hcl
variable "recordset_type" {
  description = "The type of the recordset"
  type        = string
  default     = "TXT"
}

variable "recordset_ttl" {
  description = "The time to live (TTL) of the recordset"
  type        = number
  default     = 300
}

# Create DNS recordset resource in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block), used for authorization verification
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

**Parameter Description**:
- **provider**: The provider alias, specified as `huaweicloud.domain_master` to use the master account provider
- **zone_id**: The ID of the DNS zone, obtained from the queried DNS zones data source
- **name**: The name of the recordset, constructed using the `format` function to combine the host from authorization resource with the main domain name
- **type**: The type of the recordset, assigned by referencing the input variable `recordset_type`, default is "TXT"
- **ttl**: The time to live of the recordset, assigned by referencing the input variable `recordset_ttl`, default is 300 seconds
- **records**: The record values, obtained from the authorization resource, wrapped in quotes for TXT records
- **provisioner**: A local-exec provisioner that waits 10 seconds after creating the recordset to ensure DNS propagation

> Note: The recordset is created in the master account to verify the authorization. The provisioner ensures sufficient time for DNS propagation before proceeding to verification.

### 6. Create DNS Zone Authorization Verify Resource

Add the following script to the TF file to instruct Terraform to create DNS zone authorization verification resources:

```hcl
# Create DNS zone authorization verify resource in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block), used to verify the authorization
resource "huaweicloud_dns_zone_authorization_verify" "test" {
  depends_on = [huaweicloud_dns_recordset.test]

  authorization_id = huaweicloud_dns_zone_authorization.test.id
}
```

**Parameter Description**:
- **depends_on**: Explicit dependency declaration, ensuring that the recordset is created before verifying the authorization
- **authorization_id**: The ID of the DNS zone authorization resource, referenced from the authorization resource created earlier

> Note: The authorization verification checks whether the TXT record created in the master account matches the expected value. Only after successful verification can the target account create the subdomain zone.

### 7. Create DNS Public Zone Resource

Add the following script to the TF file to instruct Terraform to create DNS public zone resources:

```hcl
# Create DNS public zone resource in the specified region (when the region parameter is omitted, it defaults to inheriting the region specified in the current provider block)
resource "huaweicloud_dns_zone" "test" {
  depends_on = [huaweicloud_dns_zone_authorization_verify.test]

  name      = format("%s.%s", var.sub_domain_prefix, try(data.huaweicloud_dns_zones.test.zones[0].name, "master_domain_not_found"))
  zone_type = "public"
}
```

**Parameter Description**:
- **depends_on**: Explicit dependency declaration, ensuring that the authorization is verified before creating the zone
- **name**: The name of the DNS zone, constructed using the `format` function to combine the subdomain prefix with the main domain name
- **zone_type**: The type of the DNS zone, set to "public" to create a public zone

> Note: The DNS zone is created in the target account context after successful authorization verification. The zone name must match the authorized subdomain name.

### 8. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Target account authentication configuration
target_account_access_key = "access_key_of_target_account"
target_account_secret_key = "secret_key_of_target_account"

# DNS domain configuration
main_domain_name  = "domain_name_of_target_account"
sub_domain_prefix = "dev"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other names, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values through the following methods:

1. Command line parameters: `terraform apply -var="main_domain_name=example.com" -var="sub_domain_prefix=dev"`
2. Environment variables: `export TF_VAR_main_domain_name=example.com`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 9. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating DNS zone authorization, recordset, authorization verification, and public zone
4. Run `terraform show` to view the created DNS resource details

## Reference Information

- [Huawei Cloud DNS Product Documentation](https://support.huaweicloud.com/intl/en-us/dns/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For DNS Public Zone Cross Accounts](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/dns/public-zone-cross-accounts)
