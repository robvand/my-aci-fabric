# ACI Fabric Configuration

> ⚠️ **Warning**: This repository contains the actual configuration for a specific ACI fabric (amslab-dmz) and is provided as a **reference example only**. Do not deploy this configuration to your own fabric without first reviewing and modifying all YAML files to match your environment. The IP addresses, VLAN IDs, tenant names, and policies are specific to this lab environment.

This repository demonstrates an infrastructure-as-code approach for managing a Cisco ACI fabric using the [Network as Code (NAC)](https://netascode.cisco.com/docs/data_models/apic/overview/) data model and the [NAC ACI Terraform module](https://registry.terraform.io/modules/netascode/nac-aci/aci/latest).

## Table of Contents

- [Quick Start](#quick-start)
- [Repository Structure](#repository-structure)
- [Design Principles](#design-principles)
- [Tenants](#tenants)
- [Data Model](#data-model)
- [Usage](#usage)
- [How It Works](#how-it-works)
- [CI/CD Integration](#cicd-integration)
- [References](#references)

## Quick Start

```bash
# 1. Update credentials in terraform.tfvars
cd foundation
vi terraform.tfvars  # Set apic_username, apic_password, apic_url

# 2. Deploy foundation first (required before any tenant)
terraform init && terraform apply

# 3. Deploy a tenant (update its terraform.tfvars first)
cd ../tn-edge
vi terraform.tfvars
terraform init && terraform apply
```

> **Important**: Always deploy `foundation` before any tenant folders. Foundation creates shared resources (VLAN pools, domains, interface policies) that tenants depend on.

## Repository Structure

```
├── foundation/              # Fabric-wide foundational configuration
│   ├── main.tf
│   ├── terraform.tfvars
│   ├── variables.tf
│   └── data/
│       ├── 1_pools.nac.yaml         # VLAN pools
│       ├── 2_domains.nac.yaml       # Physical and routed domains
│       ├── 3_policies.nac.yaml      # Access policies
│       ├── 4_policygroups.nac.yaml  # Interface policy groups
│       ├── 5_interfaces.nac.yaml    # Interface profiles and selectors
│       ├── fabric_policies.nac.yaml # Fabric-wide policies (BGP, AAA, etc.)
│       ├── node_policies.nac.yaml   # Node-level policies
│       ├── tn-infraservices.nac.yaml # Infrastructure services tenant
│       └── tn-mgmt.nac.yaml         # Management tenant
│
├── tn-<tenant-name>/        # Per-tenant configuration folders
│   ├── main.tf
│   ├── terraform.tfvars
│   ├── variables.tf
│   └── data/
│       └── tn-<tenant-name>.nac.yaml
```

## Design Principles

### Separation of Concerns

The configuration is split into isolated Terraform workspaces:

- **Foundation**: Manages all fabric-level resources including:
  - Access policies (VLAN pools, domains, interface policies)
  - Fabric policies (BGP, AAA, global settings)
  - Node policies
  - Interface profiles and selectors
  - Shared infrastructure tenants (mgmt, infraservices)

- **Tenant Folders** (`tn-*`): Each tenant has its own isolated folder containing only tenant-specific configuration:
  - VRFs
  - Bridge domains and subnets
  - Application profiles and endpoint groups
  - Contracts and filters
  - L3Outs (tenant-specific)

### Benefits of This Structure

1. **Isolation**: Changes to one tenant don't affect others
2. **Blast Radius**: Failed deployments are contained to a single tenant
3. **Parallel Execution**: Multiple tenants can be deployed simultaneously
4. **RBAC**: Different teams can manage different tenants
5. **State Management**: Smaller Terraform state files per workspace

## Tenants

| Folder | Description |
|--------|-------------|
| `tn-baelen` | Baelen workloads |
| `tn-cl-tal-n` | Cisco Live Talos clusters (1-30) |
| `tn-eal-certification` | EAL certification environment |
| `tn-edge` | Unified edge workloads |
| `tn-flexpod` | FlexPod |
| `tn-nutanix` | Nutanix HCI integration |
| `tn-titansphere` | TitanSphere platform |
| `tn-ts-ocp-1` | TitanSphere OpenShift cluster 1 |
| `tn-ts-tal-1` | TitanSphere Talos cluster 1 |

## Data Model

All configuration follows the Network as Code YAML data model. The YAML files in the `data/` directories define the intended state of the ACI fabric.

### Example Tenant Configuration

```yaml
apic:
  tenants:
    - name: example
      vrfs:
        - name: example.vrf-01
      bridge_domains:
        - name: 10.1.1.0-24
          vrf: example.vrf-01
          subnets:
            - ip: 10.1.1.1/24
      application_profiles:
        - name: network-segments
          endpoint_groups:
            - name: web
              bridge_domain: 10.1.1.0-24
              physical_domains:
                - example
```

## Usage

### Prerequisites

- Terraform >= 1.0
- Access to APIC with appropriate credentials

### Deployment Order

1. **Foundation first**: Deploy the `foundation` folder before any tenants. It creates shared resources like VLAN pools, domains, and interface policies that tenants reference.

2. **Tenants**: After foundation is deployed, tenants can be deployed in any order and independently of each other.

### Deploying Foundation

```bash
cd foundation
terraform init
terraform plan
terraform apply
```

### Deploying a Tenant

```bash
cd tn-<tenant-name>
terraform init
terraform plan
terraform apply
```

### Credentials

Update the `terraform.tfvars` file in each folder with your APIC credentials:

```hcl
apic_username = "admin"
apic_password = "your-password"
apic_url      = "https://apic.example.com"
```

> **Note**: The `terraform.tfvars` file contains sensitive credentials. Ensure it is listed in `.gitignore` and never committed to version control.

**Alternative**: Use environment variables instead:

```bash
export TF_VAR_apic_username="admin"
export TF_VAR_apic_password="your-password"
export TF_VAR_apic_url="https://apic.example.com"
```

> **Tip**: For team environments, configure a [remote backend](https://developer.hashicorp.com/terraform/language/backend) (e.g., Terraform Cloud, S3, or GitLab-managed state) to share state and enable locking.

## How It Works

The key difference between foundation and tenant folders is which resource types they manage. This is controlled by the `manage_*` flags in the NAC module configuration.

### Foundation (`main.tf`)

The foundation workspace manages all fabric-level policies. All `manage_*` flags are enabled:

```hcl
module "aci" {
  source  = "netascode/nac-aci/aci"

  yaml_directories = ["data"]

  manage_access_policies    = true
  manage_fabric_policies    = true
  manage_pod_policies       = true
  manage_node_policies      = true
  manage_interface_policies = true
  manage_tenants            = true
}
```

### Tenants (`main.tf`)

Tenant workspaces only manage tenant configuration. All flags are disabled except `manage_tenants`:

```hcl
module "aci" {
  source  = "netascode/nac-aci/aci"

  yaml_directories = ["data"]

  manage_access_policies    = false
  manage_fabric_policies    = false
  manage_pod_policies       = false
  manage_node_policies      = false
  manage_interface_policies = false
  manage_tenants            = true
}
```

This separation ensures tenants cannot accidentally modify fabric-level policies and keeps each Terraform state file focused on a specific scope.

## CI/CD Integration

This repository structure is designed for CI/CD automation with GitHub Actions, GitLab CI, or any pipeline tool that supports Terraform.

### Key Concepts

1. **Parameterized Pipelines**: Create a pipeline that accepts a `target_folder` parameter (e.g., `foundation`, `tn-edge`, `tn-nutanix`). The pipeline changes into that directory and runs `terraform init`, `plan`, and `apply`.

2. **Credential Management**: Store APIC credentials as pipeline secrets/variables and expose them as `TF_VAR_apic_username`, `TF_VAR_apic_password`, and `TF_VAR_apic_url` environment variables.

3. **Change Detection**: Optionally, detect which folders changed in a commit using `git diff` and only trigger deployments for affected tenants.

4. **Matrix/Parallel Execution**: Use your CI/CD platform's matrix or parallel job features to deploy multiple tenants simultaneously.

### Benefits

- **Self-service deployments**: Tenant owners can trigger their own pipelines
- **Automated deployments**: Deploy automatically on merge to main branch
- **Selective execution**: Only deploy what changed
- **Audit trail**: All changes tracked through version control and pipeline logs

## References

- [Network as Code Documentation](https://netascode.cisco.com/docs/data_models/apic/overview/)
- [NAC ACI Terraform Module](https://registry.terraform.io/modules/netascode/nac-aci/aci/latest)
