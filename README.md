# Tunnel Configuration - VXLAN, BIRD, and Calico

This Ansible playbook sets up network tunnels and multi-cluster connectivity using VXLAN tunnels, BIRD BGP routing, and Calico integration.

## Features

- **VXLAN Tunnels**: Virtual Extensible LAN tunnels for Layer 2 connectivity across networks
- **BGP Routing**: BIRD-based dynamic routing with support for ECMP and path isolation
- **Multi-Cluster Kubernetes**: Calico integration for cross-cluster pod and service connectivity
- **Dynamic Interface Configuration**: Support for any interface naming convention (eth0, ens3, etc.)
- **Comprehensive Network Tuning**: Extensive sysctl configuration for optimal performance
- **ECMP Support**: Equal-cost multipath routing for load balancing

## Project Structure

The playbook consists of three main components:

1. **VXLAN Role**: Creates VXLAN tunnels between virtual machines
2. **BIRD Role**: Configures BGP routing for dynamic route propagation
3. **Calico Role**: Integrates Kubernetes clusters via BGP peering

## Configuration

### Inventory Configuration

Each host in a VXLAN tunnel group requires:

```yaml
all:
  hosts:
    machine1:
      ansible_host: your-server-ip
      tunnel_type: src
      public_ip: your-public-ip
      physical_interfaces:
        - name: eth0
          ip: 192.168.1.10/24
    engine1:
      ansible_host: peer-server-ip
      tunnel_type: dest
      public_ip: peer-public-ip
      physical_interfaces:
        - name: ens3
          ip: 10.0.0.5/24
  children:
    vxlan_all:
      hosts:
        machine1:
        engine1:
```

### Global Variables

In `group_vars/all/tunnel.yaml`:

```yaml
# VXLAN configuration
vxlan_ip_range: 192.168.234
vxlan_cidr: 24

# ECMP and BGP settings
enable_ecmp: true
enable_path_isolation: true
```

## Usage

1. **Configure your inventory** in `inventory/your-environment/hosts.yml`
2. **Update global variables** in `group_vars/all/tunnel.yaml` and `group_vars/all/kube.yaml`
3. **Run the playbook**:

```bash
# Deploy everything
ansible-playbook -i inventory/your-environment/hosts.yml site.yml

# Deploy only VXLAN tunnels
ansible-playbook -i inventory/your-environment/hosts.yml site.yml --tags vxlan

# Deploy only BIRD BGP
ansible-playbook -i inventory/your-environment/hosts.yml site.yml --tags bird

# Deploy only Calico configuration
ansible-playbook -i inventory/your-environment/hosts.yml site.yml --tags calico
```

## Roles Overview

### VXLAN Role

Creates VXLAN tunnels with comprehensive sysctl tuning for optimal performance. See [roles/vxlan/README.md](roles/vxlan/README.md) for details.

### BIRD Role

Configures BGP routing with support for ECMP and path isolation. See [roles/bird/README.md](roles/bird/README.md) for details.

### Calico Role

Integrates Kubernetes clusters via BGP peering. See [roles/calico/README.md](roles/calico/README.md) for details.

## Troubleshooting

### Check VXLAN Tunnel Status

```bash
# View tunnel configuration
ip tunnel show

# Check VXLAN interface
ip addr show vxlan-eth0

# Test connectivity
ping 192.168.234.2  # Replace with peer tunnel IP
```

### Check BIRD BGP Status

```bash
# View BGP protocols
sudo birdc show protocols

# View BGP routes
sudo birdc show route
```

### Check Calico Status

```bash
# Check BGP peers
calicoctl node status

# View routes
calicoctl get bgp routes
```

### Logs

```bash
# VXLAN/netplan logs
journalctl -u networking

# BIRD logs
journalctl -u bird

# Calico logs (on Kubernetes nodes)
kubectl logs -n kube-system -l k8s-app=calico-node
```

## Documentation

For detailed documentation on each role:

- [VXLAN Role](roles/vxlan/README.md) - VXLAN tunnel configuration
- [BIRD Role](roles/bird/README.md) - BGP routing configuration
- [Calico Role](roles/calico/README.md) - Kubernetes multi-cluster integration
- [ECMP Role](roles/ecmp/README.md) - ECMP routing support
- [Workers ECMP Role](roles/workers-ecmp/README.md) - ECMP on Kubernetes workers
- [Dynamic Route Role](roles/dynamic-route/README.md) - Dynamic routing for clients
- [Metrics Role](roles/metrics/README.md) - Tunnel monitoring
