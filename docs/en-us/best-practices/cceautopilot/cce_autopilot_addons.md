# Deploy CCE Autopilot Addon

## Application Scenario

Cloud Container Engine (CCE) Autopilot is a serverless Kubernetes cluster form, allowing you to run containerized applications without managing nodes. In real-world scenarios, after a cluster is created, add-ons such as log collection and monitoring usually need to be installed to meet observability and O&M requirements.

This best practice will introduce how to use Terraform to automatically deploy a CCE Autopilot cluster and install the log-agent add-on on it. The operations include VPC and subnet creation, CCE Autopilot cluster creation, SWR organization creation, and add-on installation.

## Related Resources/Data Sources

This best practice involves the following main resources:

### Resources

- [Virtual Private Cloud (huaweicloud_vpc)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc)
- [Virtual Private Cloud Subnet (huaweicloud_vpc_subnet)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/vpc_subnet)
- [CCE Autopilot Cluster (huaweicloud_cce_autopilot_cluster)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_autopilot_cluster)
- [SWR Organization (huaweicloud_swr_organization)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/swr_organization)
- [CCE Autopilot Addon (huaweicloud_cce_autopilot_addon)](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs/resources/cce_autopilot_addon)

### Resource/Data Source Dependencies

```
huaweicloud_vpc
    └── huaweicloud_vpc_subnet
            └── huaweicloud_cce_autopilot_cluster
                    └── huaweicloud_cce_autopilot_addon

huaweicloud_swr_organization
    └── huaweicloud_cce_autopilot_addon
```

## Operation Steps

### 1. Script Preparation

Prepare the TF file (such as main.tf) for writing the current best practice script in the specified workspace, ensuring that it (or other TF files in the same directory) contains the provider version declaration and Huawei Cloud authentication information required for deploying resources.
For configuration introduction, refer to the [Preparation Before Deploying Huawei Cloud Resources](../../introductions/prepare_before_deploy.md).

### 2. Create a Virtual Private Cloud

Add the following script to the TF file (such as main.tf):

```hcl
# Create a virtual private cloud resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "vpc_name" {
  description = "The VPC name"
  type        = string
}

variable "vpc_cidr" {
  description = "The CIDR block of the VPC"
  type        = string
  default     = "192.168.0.0/16"
}

resource "huaweicloud_vpc" "test" {
  name = var.vpc_name
  cidr = var.vpc_cidr
}
```

**Parameter description**:
- **name**: The VPC name, assigned by referencing the input variable vpc_name
- **cidr**: The CIDR block of the VPC, assigned by referencing the input variable vpc_cidr

### 3. Create a Virtual Private Cloud Subnet

Add the following script to the TF file (such as main.tf):

```hcl
# Create a virtual private cloud subnet resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "subnet_name" {
  description = "The subnet name"
  type        = string
}

variable "subnet_cidr" {
  description = "The CIDR block of the subnet"
  type        = string
  default     = ""
}

variable "gateway_ip" {
  description = "The gateway IP address of the subnet"
  type        = string
  default     = ""
}

resource "huaweicloud_vpc_subnet" "test" {
  vpc_id     = huaweicloud_vpc.test.id
  name       = var.subnet_name
  cidr       = var.subnet_cidr == "" ? cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0) : var.subnet_cidr
  gateway_ip = var.gateway_ip == "" ? cidrhost(cidrsubnet(huaweicloud_vpc.test.cidr, 8, 0), 1) : var.gateway_ip
}
```

**Parameter description**:
- **vpc_id**: The ID of the VPC to which the subnet belongs, assigned by referencing the ID of the VPC resource created in the previous step
- **name**: The subnet name, assigned by referencing the input variable subnet_name
- **cidr**: The CIDR block of the subnet, assigned by referencing the input variable subnet_cidr; when the variable is empty, the subnet CIDR is automatically calculated based on the VPC CIDR
- **gateway_ip**: The gateway IP address of the subnet, assigned by referencing the input variable gateway_ip; when the variable is empty, the gateway IP is automatically calculated based on the subnet CIDR

### 4. Create a CCE Autopilot Cluster

Add the following script to the TF file (such as main.tf):

```hcl
# Create a CCE Autopilot cluster resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "cluster_name" {
  description = "The CCE autopilot cluster name"
  type        = string
}

variable "cluster_version" {
  description = "The Kubernetes version of the cluster"
  type        = string
  default     = "v1.36"
}

variable "cluster_alias" {
  description = "The alias of the cluster"
  type        = string
  default     = ""
}

variable "cluster_description" {
  description = "The description of the cluster"
  type        = string
  default     = "CCE Autopilot cluster for addon deployment"
}

variable "cluster_category" {
  description = "The cluster type. Only Turbo is supported"
  type        = string
  default     = "Turbo"
}

variable "cluster_type" {
  description = "The master node architecture. Valid values are: VirtualMachine"
  type        = string
  default     = "VirtualMachine"
}

variable "cluster_custom_san" {
  description = "The custom SAN field in the API server certificate of the cluster"
  type        = list(string)
  default     = []
}

variable "cluster_eip_id" {
  description = "The EIP ID of the cluster"
  type        = string
  default     = ""
}

variable "cluster_enable_snat" {
  description = "Whether SNAT is configured for the cluster"
  type        = bool
  default     = false
}

variable "cluster_enable_swr_image_access" {
  description = "Whether SWR image access is enabled for the cluster"
  type        = bool
  default     = true
}

variable "cluster_delete_efs" {
  description = "Whether to delete the associated EFS when deleting the cluster"
  type        = bool
  default     = false
}

variable "cluster_delete_eni" {
  description = "Whether to delete the associated ENI when deleting the cluster"
  type        = bool
  default     = false
}

variable "cluster_delete_net" {
  description = "Whether to delete the associated network resources when deleting the cluster"
  type        = bool
  default     = false
}

variable "cluster_delete_obs" {
  description = "Whether to delete the associated OBS when deleting the cluster"
  type        = bool
  default     = false
}

variable "cluster_delete_sfs_turbo" {
  description = "Whether to delete the associated SFS Turbo when deleting the cluster"
  type        = bool
  default     = false
}

variable "cluster_lts_reclaim_policy" {
  description = "The LTS reclaim policy. Valid values are: delete, retain"
  type        = string
  default     = "retain"
}

variable "cluster_enterprise_project_id" {
  description = "The ID of the enterprise project to which the cluster belongs"
  type        = string
  default     = "0"
}

variable "cluster_tags" {
  description = "The tags of the cluster"
  type        = map(string)
  default     = {}
}

resource "huaweicloud_cce_autopilot_cluster" "test" {
  name        = var.cluster_name
  flavor      = "cce.autopilot.cluster"
  version     = var.cluster_version
  alias       = var.cluster_alias
  description = var.cluster_description
  category    = var.cluster_category
  type        = var.cluster_type
  custom_san  = var.cluster_custom_san
  eip_id      = var.cluster_eip_id

  enable_snat             = var.cluster_enable_snat
  enable_swr_image_access = var.cluster_enable_swr_image_access

  delete_efs   = var.cluster_delete_efs
  delete_eni   = var.cluster_delete_eni
  delete_net   = var.cluster_delete_net
  delete_obs   = var.cluster_delete_obs
  delete_sfs30 = var.cluster_delete_sfs_turbo

  lts_reclaim_policy = var.cluster_lts_reclaim_policy

  host_network {
    vpc    = huaweicloud_vpc.test.id
    subnet = huaweicloud_vpc_subnet.test.id
  }

  container_network {
    mode = "eni"
  }

  eni_network {
    subnets {
      subnet_id = huaweicloud_vpc_subnet.test.ipv4_subnet_id
    }
  }

  extend_param {
    enterprise_project_id = var.cluster_enterprise_project_id
  }

  tags = var.cluster_tags

  lifecycle {
    ignore_changes = [
      enable_snat,
      delete_efs,
      delete_obs,
    ]
  }
}
```

**Parameter description**:
- **name**: The cluster name, assigned by referencing the input variable cluster_name
- **flavor**: The cluster flavor, fixed to cce.autopilot.cluster
- **version**: The Kubernetes version of the cluster, assigned by referencing the input variable cluster_version
- **alias**: The alias of the cluster, assigned by referencing the input variable cluster_alias
- **description**: The description of the cluster, assigned by referencing the input variable cluster_description
- **category**: The cluster type, assigned by referencing the input variable cluster_category
- **type**: The master node architecture of the cluster, assigned by referencing the input variable cluster_type
- **custom_san**: The custom SAN field in the API server certificate, assigned by referencing the input variable cluster_custom_san
- **eip_id**: The ID of the EIP bound to the cluster, assigned by referencing the input variable cluster_eip_id
- **enable_snat**: Whether SNAT is configured, assigned by referencing the input variable cluster_enable_snat
- **enable_swr_image_access**: Whether SWR image access is enabled, assigned by referencing the input variable cluster_enable_swr_image_access
- **delete_efs**, **delete_eni**, **delete_net**, **delete_obs**, **delete_sfs30**: Whether to delete the associated EFS, ENI, network resources, OBS, and SFS Turbo when deleting the cluster, assigned by referencing the input variables cluster_delete_efs, cluster_delete_eni, cluster_delete_net, cluster_delete_obs, and cluster_delete_sfs_turbo respectively
- **lts_reclaim_policy**: The LTS reclaim policy, assigned by referencing the input variable cluster_lts_reclaim_policy
- **host_network**: The host network configuration of the cluster, assigned by referencing the ID of the subnet resource created in the previous step
- **container_network**: The container network configuration, using the eni mode in this best practice
- **eni_network**: The ENI network configuration, assigned by referencing the IPv4 subnet ID of the subnet created in the previous step
- **extend_param**: The extended parameters, where the enterprise project ID is assigned by referencing the input variable cluster_enterprise_project_id
- **tags**: The tags of the cluster, assigned by referencing the input variable cluster_tags

### 5. Create a Software Repository for Container Organization

Add the following script to the TF file (such as main.tf):

```hcl
# Create a software repository for container organization resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "swr_organization_name" {
  description = "The name of the SWR organization"
  type        = string
}

resource "huaweicloud_swr_organization" "test" {
  name = var.swr_organization_name
}
```

**Parameter description**:
- **name**: The SWR organization name, assigned by referencing the input variable swr_organization_name; the name must be globally unique across Huawei Cloud

### 6. Create a CCE Autopilot Addon

Add the following script to the TF file (such as main.tf):

```hcl
# Create a CCE Autopilot addon resource in the specified region (if the region parameter is omitted, it inherits the region specified in the current provider block)
variable "addon_template_name" {
  description = "The name of the add-on template to be installed"
  type        = string
  default     = "log-agent"
}

variable "addon_version" {
  description = "The version of the add-on. If not specified, the latest compatible version will be used"
  type        = string
  default     = ""
}

variable "addon_name" {
  description = "The name of the add-on"
  type        = string
  default     = ""
}

variable "addon_alias" {
  description = "The alias of the add-on"
  type        = string
  default     = ""
}

variable "addon_values_basic" {
  description = "The basic configuration values for the add-on"
  type        = map(any)
  default     = {}
}

variable "addon_values_flavor" {
  description = "The flavor configuration values for the add-on"
  type        = map(any)
  default     = {}
}

variable "addon_values_custom" {
  description = "The custom configuration values for the add-on"
  type        = map(any)
  default     = {}
}

resource "huaweicloud_cce_autopilot_addon" "test" {
  cluster_id          = huaweicloud_cce_autopilot_cluster.test.id
  addon_template_name = var.addon_template_name
  version             = var.addon_version
  name                = var.addon_name
  alias               = var.addon_alias

  values = {
    "basic" = jsonencode(merge({
      "aomEndpoint" : "https://aom.${var.region_name}.myhuaweicloud.com",
      "iam_url" : "iam.${var.region_name}.myhuaweicloud.com",
      "ltsAccessEndpoint" : "https://lts-access.${var.region_name}.myhuaweicloud.com:8102",
      "ltsEndpoint" : "https://lts.${var.region_name}.myhuaweicloud.com",
      "region" : var.region_name,
      "swr_addr" : "swr.${var.region_name}.myhuaweicloud.com",
      "swr_user" : huaweicloud_swr_organization.test.name,
      "rbac_enabled" : true,
      "cluster_version" : huaweicloud_cce_autopilot_cluster.test.version
    }, var.addon_values_basic))

    "flavor" = jsonencode(merge({
      "category" : ["Autopilot"],
      "description" : "Applicable to clusters where the logs of a single pod are less than 10000/s or 5 MB/s.",
      "is_default" : true,
      "name" : "custom-resources",
      "resources" : [
        {
          "name" : "log-operator",
          "limitsCpu" : "1000m",
          "requestsCpu" : "1000m",
          "replicas" : 3,
          "limitsMem" : "2048Mi",
          "requestsMem" : "2048Mi"
        },
        {
          "name" : "otel-collector-event",
          "limitsCpu" : "1000m",
          "requestsCpu" : "1000m",
          "replicas" : 3,
          "limitsMem" : "2048Mi",
          "requestsMem" : "2048Mi"
        }
      ],
      "size" : "custom"
    }, var.addon_values_flavor))

    "custom" = jsonencode(merge({
      "accessKey" : "",
      "agency_name" : "",
      "aomEndpoint" : "https://aom.${var.region_name}.myhuaweicloud.com",
      "aomPrivateEndpointIP" : "",
      "bufferChunkSize" : "128k",
      "bufferMaxSize" : "512k",
      "caCert" : "",
      "clusterID" : huaweicloud_cce_autopilot_cluster.test.id,
      "clusterName" : huaweicloud_cce_autopilot_cluster.test.name,
      "cluster_category" : "CCE",
      "createAudit" : true,
      "createDefaultEvent" : true,
      "createDefaultEventToAOM" : true,
      "createDefaultStdout" : true,
      "createKubeApiserver" : false,
      "createKubeControllerManager" : false,
      "createKubeScheduler" : false,
      "enableEventReport" : true,
      "enableFullPathCollection" : false,
      "enableGcrypto" : true,
      "enableLogOperatorHA" : true,
      "enableLogReport" : true,
      "featureGates" : ["sendAOMAllEvent", "outputKafka", "podLabelExclude", "fullPathCollection", "independentEvents"],
      "host_network" : false,
      "ltsAccessEndpoint" : "https://lts-access.${var.region_name}.myhuaweicloud.com:8102",
      "ltsAuditStreamID" : "",
      "ltsEndpoint" : "https://lts.${var.region_name}.myhuaweicloud.com",
      "ltsEnterpriseProjectID" : var.cluster_enterprise_project_id,
      "ltsEventStreamID" : "",
      "ltsGroupID" : "",
      "ltsKubeApiserverStreamID" : "",
      "ltsKubeControllerManagerStreamID" : "",
      "ltsKubeSchedulerStreamID" : "",
      "ltsLogReportDomain" : "",
      "ltsPrivateEndpointIP" : "",
      "ltsStdoutStreamID" : "",
      "maxEventAgeSeconds" : "",
      "memBufLimit" : "40mb",
      "multiAZEnabled" : false,
      "otelReportLogs" : true,
      "paasakskEnable" : true,
      "podDisruptionBudget" : { "create" : true, "maxUnavailable" : 1 },
      "projectID" : "",
      "secretKey" : "",
      "securityToken" : "",
      "serverCert" : "",
      "serverKey" : ""
    }, var.addon_values_custom))
  }

  depends_on = [
    huaweicloud_cce_autopilot_cluster.test,
    huaweicloud_swr_organization.test
  ]

  lifecycle {
    ignore_changes = [
      values
    ]
  }
}
```

**Parameter description**:
- **cluster_id**: The ID of the cluster to which the add-on belongs, assigned by referencing the ID of the CCE Autopilot cluster resource created in the previous step
- **addon_template_name**: The name of the add-on template, assigned by referencing the input variable addon_template_name
- **version**: The version of the add-on, assigned by referencing the input variable addon_version; if not specified, the latest compatible version is used
- **name**: The name of the add-on, assigned by referencing the input variable addon_name
- **alias**: The alias of the add-on, assigned by referencing the input variable addon_alias
- **values**: The configuration values of the add-on, including basic, flavor, and custom parts, which are merged and assigned by referencing the input variables addon_values_basic, addon_values_flavor, and addon_values_custom respectively; the basic part references the SWR organization name and the cluster version, and the custom part references the cluster ID, cluster name, and enterprise project ID

### 7. Preset Input Parameters Required for Resource Deployment (Optional)

In this best practice, some resources use input variables to assign configuration content. These input parameters need to be manually entered during subsequent deployment.
At the same time, Terraform provides a method to preset these configurations through the `tfvars` file, which can avoid repeated input during each execution.

Create a `terraform.tfvars` file in the working directory, with the following example content:

```hcl
# Authentication variables
region_name = "cn-north-4"
access_key  = "your_access_key"
secret_key  = "your_secret_key"

# VPC and subnet variables
vpc_name    = "tf_test_vpc"
subnet_name = "tf_test_subnet"

# Cluster variables
cluster_name = "tf-test-autopilot-cluster"

# SWR organization variables
swr_organization_name = "tf-test-swr-org"

# Addon variables
addon_template_name = "log-agent"
```

**Usage**:

1. Save the above content as a `terraform.tfvars` file in the working directory (this file name allows users to automatically import the content of the `tfvars` file when executing terraform commands; for other names, `.auto` needs to be added before tfvars, such as `variables.auto.tfvars`)
2. Modify parameter values as needed
3. When executing `terraform plan` or `terraform apply`, Terraform will automatically read the variable values from this file

In addition to using the `terraform.tfvars` file, you can also set variable values in the following ways:

1. Command line parameters: `terraform apply -var="vpc_name=my-vpc"`
2. Environment variables: `export TF_VAR_vpc_name=my-vpc`
3. Custom named variable files: `terraform apply -var-file="custom.tfvars"`

> Note: If the same variable is set through multiple methods, Terraform will use variable values according to the following priority: command line parameters > variable files > environment variables > default values.

### 8. Initialize and Apply Terraform Configuration

After completing the above script configuration, execute the following steps to create resources:

1. Run `terraform init` to initialize the environment
2. Run `terraform plan` to view the resource creation plan
3. After confirming that the resource plan is correct, run `terraform apply` to start creating the CCE Autopilot cluster and its add-on
4. Run `terraform show` to view the created CCE Autopilot cluster and its add-on

## Reference Information

- [Huawei Cloud Cloud Container Engine Product Documentation](https://support.huaweicloud.com/cce/index.html)
- [Huawei Cloud Provider Documentation](https://registry.terraform.io/providers/huaweicloud/huaweicloud/latest/docs)
- [Best Practice Source Code Reference For CCE Autopilot Addon](https://github.com/huaweicloud/terraform-provider-huaweicloud/tree/master/examples/cceautopilot/cce-autopilot-addons)
