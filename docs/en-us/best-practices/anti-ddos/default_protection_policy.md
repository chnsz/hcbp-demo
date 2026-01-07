# Deploy Default Protection Policy

## Application Scenario

Anti-DDoS (Anti-Distributed Denial of Service) is a distributed denial-of-service attack protection service provided by Huawei Cloud, which can effectively protect public IPs from DDoS attacks and ensure stable business operations. The Anti-DDoS default protection policy is a global protection policy that applies to all EIPs in the account. When a DDoS attack is detected, the system will automatically start traffic cleaning based on the configured traffic threshold, filter out attack traffic, and only forward normal traffic to the origin server.

This best practice will introduce how to use Terraform to automatically deploy an Anti-DDoS default protection policy, which provides a unified protection configuration for all EIPs in your account.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Anti-DDoS Default Protection Policy Resource (huaweicloud_antiddos_default_protection_policy)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/antiddos_default_protection_policy)

### Resource/Data Source Dependencies

```
huaweicloud_antiddos_default_protection_policy
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the "Preparation Before Deploying Huawei Cloud Resources" document for configuration introduction.

### 2. Create Anti-DDoS Default Protection Policy Resource

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an Anti-DDoS default protection policy resource:

```hcl
variable "antiddos_traffic_threshold" {
  description = "The traffic cleaning threshold in Mbps"
  type        = number
}

# Create Anti-DDoS default protection policy resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_antiddos_default_protection_policy" "test" {
  traffic_threshold = var.antiddos_traffic_threshold
}
```

**Parameter Description**:
- **traffic_threshold**: The traffic cleaning threshold (unit: Mbps), assigned by referencing the input variable antiddos_traffic_threshold. When the inbound traffic of an EIP exceeds this threshold, the system will automatically start traffic cleaning.

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, the resource uses input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# Anti-DDoS Default Protection Policy Configuration
antiddos_traffic_threshold = 200
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="antiddos_traffic_threshold=200"`
2. Environment variables: `export TF_VAR_antiddos_traffic_threshold=200`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the Anti-DDoS default protection policy
4. Run `terraform show` to view the details of the created Anti-DDoS default protection policy

## Reference Information

- [Huawei Cloud Anti-DDoS Product Documentation](https://support.huaweicloud.com/antiddos/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For Anti-DDoS Default Protection Policy](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/antiddos/default-protection-policy)
