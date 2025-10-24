# VXLAN Role - Level 1: VM Connectivity with Dual Interface Support

The VXLAN role establishes Virtual Extensible LAN (VXLAN) tunnels between virtual machines, providing Layer 2 connectivity across different networks or data centers with support for multiple physical interfaces.

## Overview

This role creates VXLAN tunnels with dual interface support, allowing you to have separate VXLAN tunnels on different physical interfaces (e.g., eth0 and eth1) for redundancy and load distribution. It's designed to work with machines that have multiple network interfaces and engines with single interfaces.

## What This Role Does

1. **Installs Prerequisites**: Ensures `iproute2` package is installed
2. **Enables IP Forwarding**: Configures `net.ipv4.ip_forward=1` for packet routing
3. **Creates Dual VXLAN Tunnels**: Generates netplan configuration for VXLAN interfaces on both eth0 and eth1
4. **Configures Networking**: Applies netplan changes to activate tunnels
5. **Sets up NAT**: Configures MASQUERADE rules for 10.0.0.0/16 networks (when tunnel_type is "dest")

## Requirements

- **Operating System**: Ubuntu (uses netplan)
- **Network**: Public IP addresses for tunnel endpoints
- **Firewall**: Allow VXLAN traffic (UDP port 4789)
- **Privileges**: Root/sudo access
- **Interfaces**: Machine should have eth0 and eth1 interfaces

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

### Eth0 VXLAN Tunnels

- Host 1: `{vxlan_ip_range}.1/{vxlan_cidr}` → `192.168.234.1/24`
- Host 2: `{vxlan_ip_range}.2/{vxlan_cidr}` → `192.168.234.2/24`

### Eth1 VXLAN Tunnels

- Host 1: `{vxlan_ip_range}.101/{vxlan_cidr}` → `192.168.234.101/24`
- Host 2: `{vxlan_ip_range}.102/{vxlan_cidr}` → `192.168.234.102/24`

## VXLAN ID Assignment

VXLAN IDs are automatically assigned to avoid conflicts:

### Eth0 VXLAN IDs

- Host 1: VXLAN ID 101
- Host 2: VXLAN ID 102

### Eth1 VXLAN IDs

- Host 1: VXLAN ID 201
- Host 2: VXLAN ID 202

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

The role creates two netplan configuration files on each host:

### `/etc/netplan/90-vxlan-eth0.yaml`

```yaml
network:
  version: 2
  tunnels:
    vxlan-eth0:
      mode: vxlan
      local: 192.168.1.10 # Local eth0 interface IP
      remote: 87.247.168.200 # Peer's public IP
      id: 101 # VXLAN ID
      port: 4789 # VXLAN port
      ttl: 255
      addresses:
        - 192.168.234.1/24 # Tunnel IP
```

### `/etc/netplan/90-vxlan-eth1.yaml`

```yaml
network:
  version: 2
  tunnels:
    vxlan-eth1:
      mode: vxlan
      local: 192.168.2.10 # Local eth1 interface IP
      remote: 87.247.168.200 # Peer's public IP
      id: 201 # VXLAN ID
      port: 4789 # VXLAN port
      ttl: 255
      addresses:
        - 192.168.234.101/24 # Tunnel IP
```

## Network Architecture

### Dual Interface VXLAN Example

```
┌─────────────────────────────────┐                               ┌─────────────────┐
│           machine1              │                               │    engine1      │
│                                 │                               │                 │
│ eth0: 192.168.1.10             │    VXLAN Tunnel 1 (ID: 101)  │ Public IP:      │
│ vxlan-eth0: 192.168.234.1/24   │◄─────────────────────────────►│ 87.247.168.200  │
│                                 │                               │                 │
│ eth1: 192.168.2.10             │    VXLAN Tunnel 2 (ID: 201)  │ vxlan-ens3:     │
│ vxlan-eth1: 192.168.234.101/24 │◄─────────────────────────────►│ 192.168.234.2/24│
└─────────────────────────────────┘                               └─────────────────┘
     tunnel_type: src                                                 tunnel_type: dest
```

### Multiple Tunnels Example

```
Site A (src)                                       Site B (dest)
┌─────────────┐                                   ┌─────────────┐
│  machine1   │───── VXLAN Tunnel 1 (eth0) ─────►│   engine1   │
│192.168.234.1│                                   │192.168.234.2│
│192.168.234.101│───── VXLAN Tunnel 2 (eth1) ─────►│192.168.234.102│
└─────────────┘                                   └─────────────┘
```

## NAT Configuration

For hosts with `tunnel_type: dest` and interfaces in the 10.0.0.0/16 range, the role automatically configures MASQUERADE:

```bash
iptables -t nat -A POSTROUTING -o {main_interface} -j MASQUERADE
```

This allows traffic from the tunnel to be NAT'd when exiting through the main interface.

## Tasks Breakdown

### 1. Facts Gathering (`tasks/facts.yaml`)

Determines:

- Which VXLAN group the host belongs to (vxlan1, vxlan2, etc.)
- Who the peer host is (the other host in the same group)
- Peer type (src or dest)

### 2. VXLAN Setup (`tasks/vxlan.yaml`)

Performs:

- Package installation (iproute2)
- IP forwarding enablement
- Dual netplan configuration generation (eth0 and eth1)
- Network activation
- NAT/MASQUERADE setup (for dest hosts in 10.0.0.0/16)

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

### Check IP Forwarding

```bash
# Verify IP forwarding is enabled
sysctl net.ipv4.ip_forward

# Should return: net.ipv4.ip_forward = 1
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

### Issue: Only One Interface Working

**Symptoms**: eth0 tunnel works but eth1 doesn't (or vice versa)

**Solutions**:

```bash
# Check if eth1 interface exists and has IP
ip addr show eth1

# Verify eth1 has a valid IP address
ip -4 addr show eth1

# Check if eth1 is up
ip link show eth1

# Bring up eth1 if needed
sudo ip link set eth1 up
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

## Performance Tuning

### MTU Optimization

VXLAN adds 50 bytes of overhead. Optimize MTU to avoid fragmentation:

```yaml
# In netplan configuration
tunnels:
  vxlan-eth0:
    mtu: 1450 # 1500 - 50 bytes VXLAN overhead
  vxlan-eth1:
    mtu: 1450
```

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

- `/etc/netplan/90-vxlan-eth0.yaml` - VXLAN tunnel configuration for eth0
- `/etc/netplan/90-vxlan-eth1.yaml` - VXLAN tunnel configuration for eth1
- `/etc/sysctl.conf` - IP forwarding configuration
- `/etc/iptables/rules.v4` - NAT rules (implicitly via iptables module)

## Idempotency

This role is idempotent and can be run multiple times safely:

- Netplan configuration is only updated if changes are detected
- IP forwarding sysctl is set with `state: present`
- Package installation skips if already installed

## Related Documentation

- [Main README](../../README.md)
- [BIRD Role Documentation](../bird/README.md)
- [Calico Role Documentation](../calico/README.md)
