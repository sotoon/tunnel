# Multi-Level Network Connectivity with GRE, BGP, and Kubernetes

An Ansible-based automation system providing three levels of network connectivity: VM-to-VM tunneling, BGP route propagation, and multi-cluster Kubernetes networking.

## Overview

This playbook enables flexible network connectivity across geographically distributed or network-isolated infrastructure. You can use each level independently or combine them based on your needs:

### Three Levels of Connectivity

1. **[Level 1: GRE Tunnels](roles/gre/README.md)** - VM Connectivity

   - Establish GRE tunnels between virtual machines
   - Connect VMs across different networks or data centers
   - Use independently for simple VM-to-VM connectivity

2. **[Level 2: BIRD BGP](roles/bird/README.md)** - Route Propagation

   - Populate routes between machines using BGP
   - Dynamic routing with failover and load balancing
   - Requires Level 1 (GRE tunnels) to be configured first

3. **[Level 3: Calico BGP](roles/calico/README.md)** - Kubernetes Clusters
   - Connect multiple Kubernetes clusters
   - Enable cross-cluster pod and service communication
   - Requires Level 1 and Level 2 to be configured first

## Quick Start

### Prerequisites

- **Operating System**: Ubuntu (uses netplan and apt)
- **Ansible**: Version 2.9 or higher
- **Python**: Python 3
- **Ansible Collections**: `kubernetes.core` (for Level 3 only)
- **Access**: SSH access with sudo privileges
- **Network**: Public IPs for tunnel endpoints, GRE traffic allowed (IP protocol 47)

### Installation

1. Clone this repository:

```bash
git clone <repository-url>
cd tunnel
```

2. Install required Ansible collections (if using Level 3):

```bash
ansible-galaxy collection install kubernetes.core
```

3. Configure your inventory:

```bash
cp -r inventory/sample inventory/myenv
vim inventory/myenv/hosts.yml
```

4. Configure variables:

```bash
vim group_vars/all/tunnel.yaml
vim group_vars/all/kube.yaml  # Only for Level 3
```

### Basic Usage

**Deploy all levels:**

```bash
ansible-playbook -i inventory/sample/hosts.yml site.yml
```

**Deploy Level 1 only (GRE tunnels):**

```bash
ansible-playbook -i inventory/sample/hosts.yml site.yml --tags gre
```

**Deploy Level 1 + Level 2 (GRE + BIRD):**

```bash
ansible-playbook -i inventory/sample/hosts.yml site.yml --limit gre1,gre2,gre3
```

**Deploy Level 3 only (Calico):**

```bash
ansible-playbook -i inventory/sample/hosts.yml site.yml --tags calico
```

## Project Structure

```
.
├── README.md                          # This file
├── site.yml                           # Main playbook
├── inventory/
│   └── sample/
│       └── hosts.yml                  # Sample inventory
├── group_vars/
│   └── all/
│       ├── main.yml                   # Variable includes
│       ├── tunnel.yaml                # GRE tunnel configuration
│       └── kube.yaml                  # Kubernetes configuration
└── roles/
    ├── gre/                           # Level 1: GRE Tunnels
    │   └── README.md                  # GRE role documentation
    ├── bird/                          # Level 2: BGP Routing
    │   └── README.md                  # BIRD role documentation
    └── calico/                        # Level 3: Kubernetes
        └── README.md                  # Calico role documentation
```

## Configuration

### Inventory Setup

Define your tunnel endpoints in `inventory/<env>/hosts.yml`:

```yaml
all:
  hosts:
    # Tunnel 1
    machine1:
      ansible_host: machine-1.example.com
      tunnel_type: src
      public_ip: 87.247.187.238
    engine1:
      ansible_host: engine-1.example.com
      tunnel_type: dest
      public_ip: 87.247.168.200
    # Tunnel 2
    machine2:
      ansible_host: machine-2.example.com
      tunnel_type: src
      public_ip: 87.247.188.64
    engine2:
      ansible_host: engine-2.example.com
      tunnel_type: dest
      public_ip: 87.247.168.86
  children:
    gre1:
      hosts:
        machine1:
        engine1:
    gre2:
      hosts:
        machine2:
        engine2:
```

**Key Points**:

- Each tunnel requires exactly 2 hosts: one `src` and one `dest`
- Group tunnels using `gre1`, `gre2`, `gre3`, etc.
- `public_ip` must be reachable between tunnel endpoints

### Global Variables

Edit `group_vars/all/tunnel.yaml`:

```yaml
tunnel_ip_subnet: 172.16.234.0
tunnel_ip_range: 172.16.234
tunnel_cidr: 24
tunnel_interface_name: gre
```

### Kubernetes Configuration (Level 3 only)

Edit `group_vars/all/kube.yaml`:

```yaml
KUBECONFIG_1: /path/to/cluster1-kubeconfig.yaml
KUBECONFIG_2: /path/to/cluster2-kubeconfig.yaml
KUBE_WORKERS_1:
  - 10.0.0.243
  - 10.0.0.183
KUBE_WORKERS_2:
  - 10.0.10.27
  - 10.0.10.200
```

## Use Cases

### Use Case 1: Simple VM-to-VM Connectivity

Connect VMs across different networks without BGP or Kubernetes:

```bash
# Configure only GRE role
ansible-playbook -i inventory/sample/hosts.yml site.yml --tags gre
```

**Result**: Direct encrypted tunnel between VMs using GRE.

### Use Case 2: Multi-Site Routing with BGP

Establish dynamic routing between multiple sites:

```bash
# Configure GRE + BIRD roles
ansible-playbook -i inventory/sample/hosts.yml site.yml --limit gre1,gre2,gre3
```

**Result**: Multiple GRE tunnels with BGP-based route propagation and automatic failover.

### Use Case 3: Multi-Cluster Kubernetes

Connect multiple Kubernetes clusters for pod-to-pod communication:

```bash
# Configure all three levels
ansible-playbook -i inventory/sample/hosts.yml site.yml
```

**Result**: Full stack with GRE tunnels, BGP routing, and Kubernetes cluster networking.

## Documentation

- **[GRE Role Documentation](roles/gre/README.md)** - Level 1: Setting up GRE tunnels
- **[BIRD Role Documentation](roles/bird/README.md)** - Level 2: Configuring BGP routing
- **[Calico Role Documentation](roles/calico/README.md)** - Level 3: Connecting Kubernetes clusters

## Verification

### Verify Level 1 (GRE Tunnels)

```bash
# On tunnel endpoint
ip tunnel show
ip addr show gre
ping {remote_tunnel_ip}
```

### Verify Level 2 (BIRD BGP)

```bash
# On tunnel endpoint
sudo birdc show protocols
sudo birdc show route
```

### Verify Level 3 (Calico)

```bash
# On Kubernetes cluster
kubectl get bgppeers
calicoctl node status
```

## Troubleshooting

For detailed troubleshooting, see the individual role documentation:

- [GRE Troubleshooting](roles/gre/README.md#troubleshooting)
- [BIRD Troubleshooting](roles/bird/README.md#troubleshooting)
- [Calico Troubleshooting](roles/calico/README.md#troubleshooting)

## Security Considerations

1. **GRE Tunnels**: GRE is not encrypted by default - consider IPsec or WireGuard for encryption
2. **BGP Authentication**: Current configuration doesn't use BGP MD5 authentication
3. **Firewall**: Ensure appropriate firewall rules for GRE (protocol 47) and BGP (TCP 179, 178)
4. **Access Control**: Secure kubeconfig files and SSH keys

## Limitations

- Designed for Ubuntu-based systems (uses netplan and apt)
- Assumes two Kubernetes clusters for Level 3
- GRE tunnels are not encrypted by default
- Requires manual firewall configuration

## Contributing

[Add contribution guidelines here]

## License

[Add your license information here]

## Authors

[Add author information here]
