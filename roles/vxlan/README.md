# VXLAN Role - Level 1: VM Connectivity with Dual Interface Support

The VXLAN role establishes Virtual Extensible LAN (VXLAN) tunnels between virtual machines, providing Layer 2 connectivity across different networks or data centers with support for multiple physical interfaces.

## Overview

This role creates VXLAN tunnels with dual interface support, allowing you to have separate VXLAN tunnels on different physical interfaces (e.g., eth0 and eth1) for redundancy and load distribution. It's designed to work with machines that have multiple network interfaces and engines with single interfaces.

## What This Role Does

1. **Installs Prerequisites**: Ensures `iproute2` and `iptables-persistent` packages are installed
2. **Configures System Parameters**: Sets comprehensive sysctl parameters for optimal network performance including:
   - IP forwarding and multipath routing
   - Reverse path filtering (rp_filter)
   - Netfilter conntrack settings
   - TCP/IP memory and buffer tuning
   - Network queue limits
3. **Creates VXLAN Tunnels**: Generates netplan configuration for VXLAN interfaces with dynamic interface name support
4. **Configures Networking**: Applies netplan changes to activate tunnels
5. **Sets up NAT**: Configures MASQUERADE rules on main interface for both src and dest hosts

## Requirements

- **Operating System**: Ubuntu (uses netplan)
- **Network**: Public IP addresses for tunnel endpoints
- **Firewall**: Allow VXLAN traffic (UDP port 4789)
- **Privileges**: Root/sudo access
- **Interfaces**: Must define `physical_interfaces` in inventory with at least one interface
- **Python3**: Required for network range calculations

## Configuration

### Inventory Variables

Each host in the VXLAN tunnel requires these variables:

```yaml
all:
  hosts:
    machine1:
      ansible_host: 192.168.1.10
      tunnel_type: src # Required: "src" or "dest"
      public_ip: 87.247.187.238 # Required: Public IP for tunnel endpoint
    engine1:
      ansible_host: 10.0.0.5
      tunnel_type: dest
      public_ip: 87.247.168.200
  children:
    vxlan1: # Group name: vxlan1, vxlan2, vxlan3, etc.
      hosts:
        machine1:
        engine1:
```

### Global Variables

In `group_vars/all/tunnel.yaml`:

```yaml
# VXLAN specific configuration
vxlan_ip_subnet: 192.168.234.0 # VXLAN network base
vxlan_ip_range: 192.168.234 # First three octets
vxlan_cidr: 24 # CIDR prefix length
```

### Variable Explanations

- **`tunnel_type`**: Must be either `src` or `dest`. Each VXLAN group needs exactly one of each.
- **`public_ip`**: The publicly accessible IP address used as the tunnel endpoint
- **`ansible_host`**: The address Ansible uses to connect (can be private IP, hostname, etc.)

## Tunnel IP Assignment

VXLAN tunnel IPs are automatically assigned based on the host's index in the inventory:

- Host 1: `{vxlan_ip_range}.1/{vxlan_cidr}` → `192.168.234.1/24`
- Host 2: `{vxlan_ip_range}.2/{vxlan_cidr}` → `192.168.234.2/24`

The interface name is dynamically determined from `physical_interfaces[0].name` in the inventory.

## VXLAN ID Assignment

VXLAN IDs are automatically assigned:

- All VXLAN tunnels use VXLAN ID 101

## Usage

### Deploy VXLAN Tunnels Only

```bash
ansible-playbook -i inventory/cafe/hosts.yml site.yml --tags vxlan
```

### Deploy Specific Tunnel Group

```bash
ansible-playbook -i inventory/cafe/hosts.yml site.yml --limit vxlan1 --tags vxlan
```

### Update Existing Tunnel Configuration

```bash
ansible-playbook -i inventory/cafe/hosts.yml site.yml --tags vxlan
```

## Generated Configuration

The role creates netplan configuration files based on the physical interface name:

### `/etc/netplan/90-vxlan-{interface_name}.yaml`

Example for eth0:

```yaml
network:
  version: 2
  tunnels:
    vxlan-eth0:
      mode: vxlan
      local: 192.168.1.10 # Local physical interface IP
      remote: 87.247.168.200 # Peer's public IP
      id: 101 # VXLAN ID
      port: 4789 # VXLAN port
      ttl: 255
      mtu: 1450
      addresses:
        - 192.168.234.1/24 # Tunnel IP
      routes:
        - to: 10.0.0.0/24 # Peer network range
          via: 192.168.234.2 # Remote tunnel IP
          mtu: 1450
```

The interface name is dynamically determined from `physical_interfaces[0].name` in the inventory.

## Network Architecture

### Single VXLAN Tunnel Example

```
┌─────────────────────────────────┐                               ┌─────────────────┐
│           machine1              │                               │    engine1      │
│                                 │                               │                 │
│ eth0: 192.168.1.10             │    VXLAN Tunnel (ID: 101)    │ Public IP:      │
│ vxlan-eth0: 192.168.234.1/24   │◄─────────────────────────────►│ 87.247.168.200  │
│                                 │                               │                 │
│                                 │                               │ ens3: 10.0.0.5  │
│                                 │                               │ vxlan-eth0:     │
│                                 │                               │ 192.168.234.2/24│
└─────────────────────────────────┘                               └─────────────────┘
     tunnel_type: src                                                 tunnel_type: dest
```

The interface names (eth0, ens3, etc.) are dynamically determined from the inventory configuration.

## NAT Configuration

The role automatically configures MASQUERADE rules on the main interface for both `src` and `dest` hosts:

```bash
iptables -t nat -A POSTROUTING -o {main_interface} -j MASQUERADE
```

The main interface is dynamically determined from the default route using `ip route show default`.

## Tasks Breakdown

### 1. Facts Gathering (`tasks/facts.yaml`)

Determines:

- Which VXLAN group the host belongs to (vxlan1, vxlan2, etc.)
- Who the peer host is (the other host in the same group)
- Peer type (src or dest)

### 2. VXLAN Setup (`tasks/vxlan.yaml`)

Performs:

- Package installation (iproute2, iptables-persistent)
- Comprehensive sysctl configuration (see Sysctl Configuration section)
- Dynamic interface name detection from inventory
- Netplan configuration generation with dynamic interface names
- Network activation
- NAT/MASQUERADE setup on main interface for both src and dest hosts

## Verification

### Check Tunnel Status

```bash
# View tunnel configuration
ip tunnel show

# Check tunnel interfaces
ip addr show vxlan-eth0
ip addr show vxlan-eth1

# View tunnel statistics
ip -s tunnel show
```

### Test Connectivity

```bash
# Ping peer through eth0 tunnel
ping 192.168.234.2  # Replace with peer's eth0 tunnel IP

# Ping peer through eth1 tunnel
ping 192.168.234.102  # Replace with peer's eth1 tunnel IP

# Test with specific packet size
ping -s 1400 192.168.234.2
ping -s 1400 192.168.234.102

# Continuous ping
ping -i 0.5 192.168.234.2
```

### Check Sysctl Configuration

```bash
# Verify IP forwarding is enabled
sysctl net.ipv4.ip_forward
# Should return: net.ipv4.ip_forward = 1

# Verify multipath routing settings
sysctl net.ipv4.fib_multipath_hash_policy
sysctl net.ipv4.fib_multipath_use_neigh

# Verify rp_filter settings (should be 0 for VXLAN)
sysctl net.ipv4.conf.all.rp_filter
sysctl net.ipv4.conf.default.rp_filter
sysctl net.ipv4.conf.{interface_name}.rp_filter
sysctl net.ipv4.conf.vxlan-{interface_name}.rp_filter

# Check conntrack settings
sysctl net.netfilter.nf_conntrack_max
sysctl net.netfilter.nf_conntrack_tcp_timeout_established

# Verify TCP/IP buffer settings
sysctl net.core.rmem_max net.core.wmem_max
sysctl net.ipv4.tcp_rmem net.ipv4.tcp_wmem
```

### Check NAT Rules

```bash
# View NAT table (for dest hosts)
sudo iptables -t nat -L POSTROUTING -v -n

# Check if MASQUERADE rule exists
sudo iptables -t nat -L POSTROUTING -v -n | grep MASQUERADE
```

## Troubleshooting

### Issue: Tunnel Not Appearing

**Symptoms**: `ip tunnel show` doesn't show the VXLAN interfaces

**Solutions**:

```bash
# Check netplan configuration
cat /etc/netplan/90-vxlan-eth0.yaml
cat /etc/netplan/90-vxlan-eth1.yaml

# Manually apply netplan
sudo netplan apply

# Check for netplan errors
sudo netplan --debug apply

# View system logs
sudo journalctl -xe
```

### Issue: Cannot Ping Peer

**Symptoms**: Tunnel exists but ping fails

**Solutions**:

```bash
# 1. Check if remote public IP is reachable
ping {peer_public_ip}

# 2. Verify VXLAN traffic is allowed (UDP port 4789)
sudo tcpdump -i any port 4789

# 3. Check routing table
ip route show

# 4. Verify firewall rules
sudo iptables -L -v -n

# 5. Check if interfaces are UP
ip link show vxlan-eth0
ip link show vxlan-eth1
sudo ip link set vxlan-eth0 up
sudo ip link set vxlan-eth1 up
```

### Issue: Interface Name Mismatch

**Symptoms**: VXLAN interface not created or wrong interface name

**Solutions**:

```bash
# Check physical interface name from inventory
# Verify it matches actual interface on the host

# List available interfaces
ip addr show

# Check physical_interfaces configuration in inventory
# Ensure physical_interfaces[0].name matches actual interface name
```

### Issue: IP Forwarding Not Working

**Symptoms**: Can ping tunnel but cannot route through it

**Solutions**:

```bash
# Enable IP forwarding manually
sudo sysctl -w net.ipv4.ip_forward=1

# Make it persistent
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf

# Reload sysctl
sudo sysctl -p
```

### Issue: MTU Problems

**Symptoms**: Some packets work, large packets fail

**Solutions**:

```bash
# Check current MTU
ip addr show vxlan-eth0
ip addr show vxlan-eth1

# Set lower MTU (accounting for VXLAN overhead)
sudo ip link set vxlan-eth0 mtu 1400
sudo ip link set vxlan-eth1 mtu 1400

# Update netplan config to include MTU
# Add under tunnel configuration:
#   mtu: 1400
```

### Issue: Firewall Blocking VXLAN

**Symptoms**: Tunnel configured but no connectivity

**Solutions**:

```bash
# Allow VXLAN traffic (UDP port 4789)
sudo iptables -A INPUT -p udp --dport 4789 -j ACCEPT
sudo iptables -A OUTPUT -p udp --dport 4789 -j ACCEPT

# For UFW users
sudo ufw allow 4789/udp

# Check if packets are being dropped
sudo iptables -L -v -n | grep DROP
```

## Security Considerations

### VXLAN Limitations

- **No Encryption**: VXLAN does not encrypt traffic by default
- **No Authentication**: VXLAN endpoints are not authenticated
- **Broadcast Domain**: VXLAN creates a Layer 2 broadcast domain

### Recommendations

1. **Use with IPsec**: Combine VXLAN with IPsec for encryption
2. **Firewall Rules**: Restrict VXLAN traffic to known peer IPs
3. **Private Networks**: Use VXLAN over VPNs or private networks when possible
4. **Monitoring**: Monitor tunnel traffic for anomalies

### Firewall Example

```bash
# Only allow VXLAN from specific peer
sudo iptables -A INPUT -p udp --dport 4789 -s {peer_public_ip} -j ACCEPT
sudo iptables -A INPUT -p udp --dport 4789 -j DROP
```

## Sysctl Configuration

The role configures comprehensive sysctl parameters for optimal VXLAN performance:

### IP Forwarding and Multipath Routing

- `net.ipv4.ip_forward = 1` - Enables IP packet forwarding
- `net.ipv4.fib_multipath_hash_policy = 1` - Uses Layer 4 (port) information for multipath hashing
- `net.ipv4.fib_multipath_use_neigh = 1` - Uses neighbor status for multipath selection

### Reverse Path Filtering

- `net.ipv4.conf.all.rp_filter = 0` - Disables rp_filter for all interfaces
- `net.ipv4.conf.default.rp_filter = 0` - Disables rp_filter for default interface
- `net.ipv4.conf.{interface_name}.rp_filter = 0` - Disables rp_filter for physical interface (dynamic)
- `net.ipv4.conf.vxlan-{interface_name}.rp_filter = 0` - Disables rp_filter for VXLAN interface (dynamic)

### Netfilter Conntrack

- `net.netfilter.nf_conntrack_max = 1048576` - Maximum number of connection tracking entries
- `net.netfilter.nf_conntrack_tcp_timeout_established = 7200` - TCP established timeout (2 hours)
- `net.netfilter.nf_conntrack_tcp_timeout_time_wait = 30` - TCP time wait timeout
- `net.netfilter.nf_conntrack_tcp_timeout_close_wait = 60` - TCP close wait timeout
- `net.netfilter.nf_conntrack_tcp_timeout_fin_wait = 30` - TCP fin wait timeout
- `net.netfilter.nf_conntrack_tcp_timeout_syn_recv = 30` - TCP SYN receive timeout
- `net.netfilter.nf_conntrack_generic_timeout = 120` - Generic connection timeout

### TCP/IP Memory and Buffers

- `net.ipv4.ip_local_port_range = 1024 65535` - Local port range
- `net.ipv4.tcp_tw_reuse = 1` - Reuse TIME_WAIT sockets
- `net.core.rmem_max = 16777216` - Maximum receive buffer size (16MB)
- `net.core.wmem_max = 16777216` - Maximum write buffer size (16MB)
- `net.core.rmem_default = 262144` - Default receive buffer size (256KB)
- `net.core.wmem_default = 262144` - Default write buffer size (256KB)
- `net.ipv4.tcp_rmem = 4096 131072 16777216` - TCP receive buffer min/default/max
- `net.ipv4.tcp_wmem = 4096 131072 16777216` - TCP write buffer min/default/max
- `net.ipv4.tcp_mem = 1048576 1572864 2097152` - TCP memory pressure thresholds

### Network Queue Limits

- `net.core.netdev_max_backlog = 250000` - Maximum number of packets queued on input
- `net.core.somaxconn = 65535` - Maximum number of pending connection requests
- `net.ipv4.tcp_max_syn_backlog = 65535` - Maximum number of SYN backlog requests

All sysctls are persisted to `/etc/sysctl.conf` and applied immediately.

## Performance Tuning

### MTU Optimization

VXLAN adds 50 bytes of overhead. The role configures MTU 1450 by default to avoid fragmentation.

### TCP MSS Clamping

```bash
# Clamp TCP MSS to avoid fragmentation
sudo iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```

## Advanced Configuration

### Custom VXLAN ID

Modify the template to change VXLAN ID assignment:

```yaml
# In vxlan.yaml task
vxlan_id: "{{ 1000 + (groups['all'] | list).index(inventory_hostname) }}"
```

### Custom Port

Modify the template to change VXLAN port:

```yaml
# In 90-vxlan.yaml.j2
port: 8472 # Default is 4789
```

## Integration with Other Roles

This role is standalone but provides the foundation for:

- **[BIRD Role](../bird/README.md)**: Uses VXLAN tunnels for BGP peering
- **[Calico Role](../calico/README.md)**: Requires VXLAN + BIRD for multi-cluster networking

## Files Modified

- `/etc/netplan/90-vxlan-{interface_name}.yaml` - VXLAN tunnel configuration (interface name is dynamic)
- `/etc/sysctl.conf` - Comprehensive sysctl configuration (see Sysctl Configuration section)
- `/etc/iptables/rules.v4` - NAT rules (via netfilter-persistent)

## Idempotency

This role is idempotent and can be run multiple times safely:

- Netplan configuration is only updated if changes are detected
- IP forwarding sysctl is set with `state: present`
- Package installation skips if already installed

## Related Documentation

- [Main README](../../README.md)
- [BIRD Role Documentation](../bird/README.md)
- [Calico Role Documentation](../calico/README.md)
