# GRE Role - Level 1: VM Connectivity

The GRE role establishes Generic Routing Encapsulation (GRE) tunnels between virtual machines, providing Layer 3 connectivity across different networks or data centers.

## Overview

This role can be used independently to create point-to-point tunnels between VMs without requiring BGP or Kubernetes. It's the foundation for Level 2 and Level 3 connectivity but works standalone for simple VM-to-VM connectivity scenarios.

## What This Role Does

1. **Installs Prerequisites**: Ensures `iproute2` package is installed
2. **Enables IP Forwarding**: Configures `net.ipv4.ip_forward=1` for packet routing
3. **Creates GRE Tunnels**: Generates netplan configuration for GRE interfaces
4. **Configures Networking**: Applies netplan changes to activate tunnels
5. **Sets up NAT**: Configures MASQUERADE rules for 10.0.0.0/16 networks (when tunnel_type is "dest")

## Requirements

- **Operating System**: Ubuntu (uses netplan)
- **Network**: Public IP addresses for tunnel endpoints
- **Firewall**: Allow GRE traffic (IP protocol 47)
- **Privileges**: Root/sudo access

## Configuration

### Inventory Variables

Each host in the GRE tunnel requires these variables:

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
    gre1: # Group name: gre1, gre2, gre3, etc.
      hosts:
        machine1:
        engine1:
```

### Global Variables

In `group_vars/all/tunnel.yaml`:

```yaml
tunnel_ip_subnet: 172.16.234.0 # Tunnel network base
tunnel_ip_range: 172.16.234 # First three octets
tunnel_cidr: 24 # CIDR prefix length
tunnel_interface_name: gre # Interface name (default: gre)
```

### Variable Explanations

- **`tunnel_type`**: Must be either `src` or `dest`. Each GRE group needs exactly one of each.
- **`public_ip`**: The publicly accessible IP address used as the tunnel endpoint
- **`ansible_host`**: The address Ansible uses to connect (can be private IP, hostname, etc.)

## Tunnel IP Assignment

Tunnel IPs are automatically assigned based on the host's index in the inventory:

- Host 1: `{tunnel_ip_range}.1/{tunnel_cidr}` → `172.16.234.1/24`
- Host 2: `{tunnel_ip_range}.2/{tunnel_cidr}` → `172.16.234.2/24`
- Host 3: `{tunnel_ip_range}.3/{tunnel_cidr}` → `172.16.234.3/24`
- And so on...

## Usage

### Deploy GRE Tunnels Only

```bash
ansible-playbook -i inventory/sample/hosts.yml site.yml --tags gre
```

### Deploy Specific Tunnel Group

```bash
ansible-playbook -i inventory/sample/hosts.yml site.yml --limit gre1 --tags gre
```

### Update Existing Tunnel Configuration

```bash
ansible-playbook -i inventory/sample/hosts.yml site.yml --tags gre
```

## Generated Configuration

The role creates `/etc/netplan/90-gre.yaml` on each host:

```yaml
network:
  version: 2
  tunnels:
    gre:
      mode: gre
      local: 192.168.1.10 # Local interface IP
      remote: 87.247.168.200 # Peer's public IP
      ttl: 255
      addresses:
        - 172.16.234.1/24 # Tunnel IP
```

## Network Architecture

### Single Tunnel Example

```
┌─────────────────┐                               ┌─────────────────┐
│   machine1      │                               │    engine1      │
│                 │                               │                 │
│ Public IP:      │    GRE Tunnel (Protocol 47)  │ Public IP:      │
│ 87.247.187.238  │◄─────────────────────────────►│ 87.247.168.200  │
│                 │                               │                 │
│ Tunnel IP:      │                               │ Tunnel IP:      │
│ 172.16.234.1/24 │                               │ 172.16.234.2/24 │
└─────────────────┘                               └─────────────────┘
     tunnel_type: src                                 tunnel_type: dest
```

### Multiple Tunnels Example

```
Site A (src)                                       Site B (dest)
┌─────────────┐                                   ┌─────────────┐
│  machine1   │───── GRE Tunnel 1 ──────────────►│   engine1   │
│ 172.16.234.1│                                   │172.16.234.2 │
└─────────────┘                                   └─────────────┘

┌─────────────┐                                   ┌─────────────┐
│  machine2   │───── GRE Tunnel 2 ──────────────►│   engine2   │
│ 172.16.234.3│                                   │172.16.234.4 │
└─────────────┘                                   └─────────────┘

┌─────────────┐                                   ┌─────────────┐
│  machine3   │───── GRE Tunnel 3 ──────────────►│   engine3   │
│ 172.16.234.5│                                   │172.16.234.6 │
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

- Which GRE group the host belongs to (gre1, gre2, etc.)
- Who the peer host is (the other host in the same group)
- Peer type (src or dest)

### 2. GRE Setup (`tasks/gre.yaml`)

Performs:

- Package installation (iproute2)
- IP forwarding enablement
- Netplan configuration generation
- Network activation
- NAT/MASQUERADE setup (for dest hosts in 10.0.0.0/16)

## Verification

### Check Tunnel Status

```bash
# View tunnel configuration
ip tunnel show

# Check tunnel interface
ip addr show gre

# View tunnel statistics
ip -s tunnel show
```

### Test Connectivity

```bash
# Ping peer through tunnel
ping 172.16.234.2  # Replace with peer's tunnel IP

# Test with specific packet size
ping -s 1400 172.16.234.2

# Continuous ping
ping -i 0.5 172.16.234.2
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

**Symptoms**: `ip tunnel show` doesn't show the GRE interface

**Solutions**:

```bash
# Check netplan configuration
cat /etc/netplan/90-gre.yaml

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

# 2. Verify GRE traffic is allowed
sudo tcpdump -i any proto gre

# 3. Check routing table
ip route show

# 4. Verify firewall rules
sudo iptables -L -v -n

# 5. Check if interface is UP
ip link show gre
sudo ip link set gre up
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
ip addr show gre

# Set lower MTU (accounting for GRE overhead)
sudo ip link set gre mtu 1400

# Update netplan config to include MTU
# Add under tunnel configuration:
#   mtu: 1400
```

### Issue: Firewall Blocking GRE

**Symptoms**: Tunnel configured but no connectivity

**Solutions**:

```bash
# Allow GRE protocol (protocol 47)
sudo iptables -A INPUT -p gre -j ACCEPT
sudo iptables -A OUTPUT -p gre -j ACCEPT

# For UFW users
sudo ufw allow proto gre

# Check if packets are being dropped
sudo iptables -L -v -n | grep DROP
```

## Security Considerations

### GRE Limitations

- **No Encryption**: GRE does not encrypt traffic by default
- **No Authentication**: GRE endpoints are not authenticated
- **IP Spoofing**: Vulnerable to IP spoofing attacks

### Recommendations

1. **Use with IPsec**: Combine GRE with IPsec for encryption
2. **Firewall Rules**: Restrict GRE traffic to known peer IPs
3. **Private Networks**: Use GRE over VPNs or private networks when possible
4. **Monitoring**: Monitor tunnel traffic for anomalies

### Firewall Example

```bash
# Only allow GRE from specific peer
sudo iptables -A INPUT -p gre -s {peer_public_ip} -j ACCEPT
sudo iptables -A INPUT -p gre -j DROP
```

## Performance Tuning

### MTU Optimization

GRE adds 24 bytes of overhead. Optimize MTU to avoid fragmentation:

```yaml
# In netplan configuration
tunnels:
  gre:
    mtu: 1476 # 1500 - 24 bytes GRE overhead
```

### TCP MSS Clamping

```bash
# Clamp TCP MSS to avoid fragmentation
sudo iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```

## Advanced Configuration

### Custom TTL

Modify the template to change TTL:

```yaml
# In 90-gre.yaml.j2
ttl: 64 # Default is 255
```

### Keepalive

Add keepalive to detect dead tunnels:

```yaml
# In netplan configuration (if supported)
tunnels:
  gre:
    keepalive:
      interval: 10
      count: 3
```

## Integration with Other Roles

This role is standalone but provides the foundation for:

- **[BIRD Role](../bird/README.md)**: Uses GRE tunnels for BGP peering
- **[Calico Role](../calico/README.md)**: Requires GRE + BIRD for multi-cluster networking

## Files Modified

- `/etc/netplan/90-gre.yaml` - GRE tunnel configuration
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
