# Workers ECMP Role

This role configures ECMP (Equal Cost Multi-Path) sysctl settings on Kubernetes worker nodes to enable proper load balancing across multiple BGP routes.

## What This Role Does

1. **Enables Layer 4 hashing** (`net.ipv4.fib_multipath_hash_policy=1`)
   - Uses source/destination ports in hash calculation
   - Better load distribution across ECMP paths
   - Prevents flow reordering issues

2. **Enables neighbor status checking** (`net.ipv4.fib_multipath_use_neigh=1`)
   - Uses neighbor entry status when selecting nexthop
   - Prevents sending packets to failed/dead neighbors
   - Improves failover behavior

## Requirements

- Root/sudo access on worker nodes
- Ansible `sysctl` module

## Usage

Include this role in your playbook targeting Kubernetes worker nodes:

```yaml
- hosts: kube_workers
  become: yes
  roles:
    - workers-ecmp
```

## Sysctl Settings

The role configures and persists the following sysctl settings:

- `net.ipv4.fib_multipath_hash_policy=1` - Layer 4 hashing
- `net.ipv4.fib_multipath_use_neigh=1` - Use neighbor status

Both settings are persisted to `/etc/sysctl.conf` and applied immediately.

## Verification

After running the role, verify the settings:

```bash
sysctl net.ipv4.fib_multipath_hash_policy
sysctl net.ipv4.fib_multipath_use_neigh

# Should show:
# net.ipv4.fib_multipath_hash_policy = 1
# net.ipv4.fib_multipath_use_neigh = 1
```

## Integration

This role works together with:
- **BIRD role**: Configures BGP routes with equal attributes for ECMP
- **Calico role**: Configures BGP peering between workers and tunnel endpoints

When both tunnel endpoints advertise routes with identical BGP attributes, the Linux kernel will install them as ECMP routes, and these sysctl settings ensure optimal load balancing.

