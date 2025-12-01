# Foundation Configuration

This folder contains the fabric-wide foundational configuration. The numbered YAML files in the `data/` directory establish a dependency chain that must be followed when adding new workloads.

## File Structure

```
data/
├── 1_pools.nac.yaml         # VLAN pools
├── 2_domains.nac.yaml       # Physical and routed domains
├── 3_policies.nac.yaml      # AAEPs (Attachable Access Entity Profiles)
├── 4_policygroups.nac.yaml  # Interface policy groups
├── 5_interfaces.nac.yaml    # Interface assignments
├── fabric_policies.nac.yaml # Fabric-wide settings (BGP, AAA, etc.)
├── node_policies.nac.yaml   # Node registration and policies
├── tn-infraservices.nac.yaml # Shared infrastructure services tenant
└── tn-mgmt.nac.yaml         # Management tenant
```

## Dependency Chain

When adding a new workload to the fabric, follow the numbered files in order. Each layer depends on the previous one:

```
1_pools → 2_domains → 3_policies → 4_policygroups → 5_interfaces
```

## Example: Adding a Dual-Homed (VPC) Workload

The following example demonstrates adding a new workload that is dual-homed to the fabric using Virtual Port Channel (VPC). The server connects to two leaf switches for redundancy, and LACP bonds the links into a single logical channel.

### 1. VLAN Pools (`1_pools.nac.yaml`)

Define the VLANs that will be used by your workload.

```yaml
- name: my-workload-vlans
  allocation: static
  ranges:
    - from: 100
      to: 100
      description: my workload network
```

### 2. Domains (`2_domains.nac.yaml`)

Create a physical or routed domain that references the VLAN pool.

```yaml
physical_domains:
  - name: my-workload
    vlan_pool: my-workload-vlans
```

### 3. AAEPs (`3_policies.nac.yaml`)

Create an AAEP that associates the domain with endpoint groups. This defines which VLANs are allowed on the interface.

```yaml
aaeps:
  - name: vlans-allowed-to-my-workload
    physical_domains:
      - my-workload
    endpoint_groups:
      - tenant: my-workload
        application_profile: network-segments
        endpoint_group: servers
        vlan: 100
```

### 4. Policy Groups (`4_policygroups.nac.yaml`)

Create an interface policy group that references the AAEP and defines interface settings.

```yaml
leaf_interface_policy_groups:
  - name: my-workload-vpc
    type: vpc
    aaep: vlans-allowed-to-my-workload
    link_level_policy: system-link-level-10G-auto
    lldp_policy: system-lldp-enabled
    port_channel_policy: system-lacp-active
```

### 5. Interfaces (`5_interfaces.nac.yaml`)

Assign the policy group to physical interfaces on leaf switches. For VPC, configure both leaf switches in the VPC pair.

```yaml
interface_policies:
  nodes:
    - id: 101
      interfaces:
        - port: 10
          description: connected to my-workload server eth0
          policy_group: my-workload-vpc
    - id: 102
      interfaces:
        - port: 10
          description: connected to my-workload server eth1
          policy_group: my-workload-vpc
```

## Other Files

| File | Purpose |
|------|---------|
| `fabric_policies.nac.yaml` | BGP AS, route reflectors, AAA, LDAP, NTP, DNS, SNMP |
| `node_policies.nac.yaml` | Node IDs, names, pod assignments, OOB addresses |
| `tn-infraservices.nac.yaml` | Shared services tenant (DHCP, DNS, storage networks) |
| `tn-mgmt.nac.yaml` | In-band and out-of-band management configuration |

## Adding a New Workload: Checklist

1. [ ] Add VLAN pool in `1_pools.nac.yaml`
2. [ ] Add physical/routed domain in `2_domains.nac.yaml`
3. [ ] Add AAEP in `3_policies.nac.yaml`
4. [ ] Add interface policy group in `4_policygroups.nac.yaml`
5. [ ] Assign interfaces in `5_interfaces.nac.yaml`
6. [ ] Create tenant folder (`tn-<workload>/`) with tenant configuration
7. [ ] Deploy foundation: `terraform apply`
8. [ ] Deploy tenant: `cd ../tn-<workload> && terraform apply`

## More Examples

For additional configuration patterns (access ports, port channels, routed interfaces, L3Outs, etc.), refer to the [Network as Code documentation](https://netascode.cisco.com/docs/data_models/apic/overview/).
