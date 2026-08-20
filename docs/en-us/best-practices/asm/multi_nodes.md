# Deploy Multi-Nodes Mesh

## Application Scenario

Application Service Mesh (ASM) provides a non-intrusive microservice governance solution with complete lifecycle management and traffic governance. It is compatible with the Kubernetes and Istio ecosystems, and supports capabilities such as load balancing, circuit breaking, and fault injection. It also includes built-in canary and blue-green release processes for one-stop automated release management. ASM is deeply integrated with Huawei Cloud Container Engine (CCE) to provide an out-of-the-box service mesh experience.

This best practice introduces how to use Terraform to automatically deploy an ASM mesh with multi-node installation and namespace sidecar injection, including creating a CCE namespace on demand, installing mesh components on multiple CCE nodes, and configuring sidecar injection for specified namespaces.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [CCE Namespace Resource (huaweicloud_cce_namespace)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_namespace)
- [ASM Mesh Resource (huaweicloud_asm_mesh)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/asm_mesh)

### Resource/Data Source Dependencies

```
huaweicloud_cce_namespace
    └── huaweicloud_asm_mesh
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (e.g., main.tf) in the specified workspace for writing the current best practice script, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
Refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md) document for configuration introduction.

### 2. Create CCE Namespace

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create a CCE namespace resource for ASM mesh sidecar injection:

```hcl
variable "namespaces" {
  description = "The list of existing namespace names for sidecar injection"
  type        = list(string)
  default     = []
}

variable "cluster_id" {
  description = "The ID of the CCE cluster to be associated with the ASM mesh"
  type        = string
}

variable "namespace_name" {
  description = "The name of the namespace to create"
  type        = string
  default     = ""

  validation {
    condition     = length(var.namespaces) != 0 || var.namespace_name != ""
    error_message = "The 'namespace_name' is required when 'namespaces' is not specified."
  }
}

# Create CCE namespace resource in the specified region (defaults to the region specified in the provider block when region parameter is omitted) for ASM mesh sidecar injection
resource "huaweicloud_cce_namespace" "test" {
  count = length(var.namespaces) == 0 ? 1 : 0

  cluster_id = var.cluster_id
  name       = var.namespace_name
}
```

**Parameter Description**:
- **count**: The number of namespace resources to create, used to control whether to create the namespace, only created when `var.namespaces` is empty
- **cluster_id**: The ID of the CCE cluster to which the namespace belongs, assigned by referencing the input variable cluster_id
- **name**: The namespace name, assigned by referencing the input variable namespace_name

> Note: When existing namespaces are specified through `namespaces`, a new CCE namespace will not be created; `namespace_name` is required when `namespaces` is not specified.

### 3. Create ASM Mesh

Add the following script to the TF file (e.g., main.tf) to instruct Terraform to create an ASM mesh resource:

```hcl
variable "mesh_name" {
  description = "The name of the ASM mesh"
  type        = string
}

variable "mesh_type" {
  description = "The type of the ASM mesh. Currently, only 'InCluster' is supported"
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

variable "node_ids" {
  description = "The list of node IDs where ASM mesh components will be installed"
  type        = list(string)
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
            values   = var.node_ids
          }
        }
      }

      injection {
        namespaces {
          field_selector {
            key      = "Name"
            operator = "In"
            values   = length(var.namespaces) > 0 ? var.namespaces : [huaweicloud_cce_namespace.test[0].name]
          }
        }
      }
    }
  }
}
```

**Parameter Description**:
- **name**: The name of the ASM mesh, assigned by referencing the input variable mesh_name
- **type**: The type of the ASM mesh, assigned by referencing the input variable mesh_type, defaults to "InCluster", currently only this value is supported
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
          - **values**: The value list of the selector, assigned by referencing the input variable node_ids, used to specify multiple nodes where mesh components will be installed
    - **injection**: The sidecar injection configuration block
      - **namespaces**: The sidecar injection namespaces configuration block
        - **field_selector**: The field selector configuration block
          - **key**: The key of the selector, fixed to "Name"
          - **operator**: The operator of the selector, fixed to "In"
          - **values**: The value list of the selector, using `var.namespaces` when it is not empty; otherwise referencing the name of the previously created CCE namespace resource (huaweicloud_cce_namespace.test)

> Note: Before creating a multi-nodes ASM mesh, ensure that the target CCE cluster and nodes are ready, and replace `cluster_id` and `node_ids` with actual available resource IDs.

### 4. Preset Input Parameters Required for Resource Deployment (Optional)

In this practice, some resources and data sources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through `tfvars` files, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory with the following example content:

```hcl
# ASM mesh configuration
mesh_name    = "multi-nodes-mesh"
mesh_version = "1.18.7-r7"

# CCE cluster and multiple nodes
cluster_id = "your-cce-cluster-id"
node_ids   = ["your-cce-node-1", "your-cce-node-2"]

namespace_name = "multi-nodes-test"

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

1. Command line parameters: `terraform apply -var="mesh_name=multi-nodes-mesh" -var="mesh_version=1.18.7-r7"`
2. Environment variables: `export TF_VAR_mesh_name=multi-nodes-mesh`
3. Custom named variable file: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable file > environment variables > default values.

### 5. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the multi-nodes ASM mesh
4. Run `terraform show` to view the created multi-nodes ASM mesh

## Reference Information

- [Huawei Cloud ASM Product Documentation](https://support.huaweicloud.com/asm/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For ASM Multi-Nodes Mesh](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/asm/multi-nodes)
