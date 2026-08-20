# Deploy Basic Mesh

## Application Scenario

Application Service Mesh (ASM) provides a non-intrusive microservice governance solution with complete lifecycle management and traffic governance. It is compatible with the Kubernetes and Istio ecosystems, and supports capabilities such as load balancing, circuit breaking, and fault injection. It also includes built-in canary and blue-green release processes for one-stop automated release management. ASM is deeply integrated with Huawei Cloud Container Engine (CCE) to provide an out-of-the-box service mesh experience.

This best practice introduces how to use Terraform to automatically deploy a basic ASM mesh associated with a single cluster, including specifying the mesh name, type, and version, and installing mesh components on the specified CCE cluster node.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [ASM Mesh Resource (huaweicloud_asm_mesh)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/asm_mesh)

### Resource/Data Source Dependencies

```
huaweicloud_asm_mesh
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) document for configuration introduction.

### 2. Create ASM Mesh

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an ASM mesh resource:

```hcl
variable "mesh_name" {
  description = "The name of the ASM mesh"
  type        = string
}

variable "mesh_type" {
  description = "The type of the ASM mesh"
  type        = string
  default     = "InCluster"
  nullable    = false
}

variable "mesh_version" {
  description = "The version of the ASM mesh"
  type        = string
}

variable "tags" {
  description = "The key/value pairs to associate with the ASM mesh"
  type        = map(string)
  default     = {}
}

variable "cluster_id" {
  description = "The ID of the CCE cluster to be associated with the ASM mesh"
  type        = string
}

variable "node_id" {
  description = "The ID of the node where ASM mesh components will be installed"
  type        = string
}

# Create ASM mesh resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted)
resource "huaweicloud_asm_mesh" "test" {
  name    = var.mesh_name
  type    = var.mesh_type
  version = var.mesh_version
  tags    = var.tags

  extend_params {
    clusters {
      cluster_id = var.cluster_id

      installation {
        nodes {
          field_selector {
            key      = "UID"
            operator = "In"
            values   = [var.node_id]
          }
        }
      }
    }
  }
}
```

**Parameter Description**:
- **name**: The name of the ASM mesh, assigned by referencing the input variable mesh_name
- **type**: The type of the ASM mesh, assigned by referencing the input variable mesh_type, defaults to "InCluster"
- **version**: The version of the ASM mesh, assigned by referencing the input variable mesh_version
- **tags**: The key/value pairs associated with the ASM mesh, assigned by referencing the input variable tags, defaults to an empty map
- **extend_params**: The extend parameters configuration block of the ASM mesh
  - **clusters**: The cluster information configuration block in the mesh
    - **cluster_id**: The ID of the associated CCE cluster, assigned by referencing the input variable cluster_id
    - **installation**: The mesh components installation configuration block
      - **nodes**: The mesh components installation nodes configuration block
        - **field_selector**: The field selector configuration block
          - **key**: The key of the selector, fixed to "UID"
          - **operator**: The operator of the selector, fixed to "In"
          - **values**: The value list of the selector, assigned by referencing the input variable node_id, used to specify the node where mesh components will be installed

> Note: Before creating a basic ASM mesh, ensure that the target CCE cluster and node are ready, and replace `cluster_id` and `node_id` with actual available resource IDs.

### 3. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# ASM mesh configuration
mesh_name    = "basic-mesh"
mesh_version = "1.18.7-r7"

# CCE cluster and node
cluster_id = "your-cce-cluster-id"
node_id    = "your-cce-cluster-node-id"

# Tags for the mesh
tags = {
  foo = "bar"
  key = "value"
}
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this filename allows users to automatically import the content of this `tfvars` file when executing terraform commands. For other naming, you need to add `.auto` before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values according to actual needs
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values in this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="mesh_name=basic-mesh" -var="mesh_version=1.18.7-r7"`
2. Environment variables: `export TF_VAR_mesh_name=basic-mesh`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 4. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the ASM mesh
4. Run `terraform show` to view the created ASM mesh

## Reference Information

- [Huawei Cloud ASM Product Documentation](https://support.huaweicloud.com/asm/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For ASM Basic Mesh](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/asm/basic)
