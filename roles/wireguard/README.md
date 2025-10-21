# WireGuard Role - Level 1: VM Connectivity

The WireGuard role establishes secure WireGuard VPN tunnels between virtual machines, providing encrypted Layer 3 connectivity across different networks or data centers.

## Overview

This role can be used independently to create secure point-to-point tunnels between VMs without requiring BGP or Kubernetes. It's the foundation for Level 2 and Level 3 connectivity but works standalone for simple VM-to-VM connectivity scenarios with strong encryption.

## What This Role Does

1. **Installs Prerequisites**: Ensures `wireguard` package is installed
2. **Enables IP Forwarding**: Configures `net.ipv4.ip_forward=1` for packet routing
3. **Generates Keys**: Creates private/public key pairs for each host
4. **Creates WireGuard Tunnels**: Generates WireGuard configuration files
5. **Configures Networking**: Applies WireGuard configuration to activate tunnels
6. **Sets up NAT**: Configures MASQUERADE rules for 10.0.0.0/16 networks (when tunnel_type is "dest")

## Requirements

- **Operating System**: Ubuntu (uses systemd and netplan)
- **Network**: Public IP addresses for tunnel endpoints
- **Firewall**: Allow UDP traffic on WireGuard port (default 51820)
- **Privileges**: Root/sudo access

## Configuration

### Inventory Variables

Each host in the WireGuard tunnel requires these variables:

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
    wireguard1: # Group name: wireguard1, wireguard2, wireguard3, etc.
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
tunnel_interface_name: wg0 # Interface name (default: wg0)
wireguard_port: 51820 # WireGuard listening port
wireguard_private_key_path: /etc/wireguard/private_key
wireguard_public_key_path: /etc/wireguard/public_key
```

### Variable Explanations

- **`tunnel_type`**: Must be either `src` or `dest`. Each WireGuard group needs exactly one of each.
- **`public_ip`**: The publicly accessible IP address used as the tunnel endpoint
- **`ansible_host`**: The address Ansible uses to connect (can be private IP, hostname, etc.)

## Tunnel IP Assignment

Tunnel IPs are automatically assigned based on the host's index in the inventory:

- Host 1: `{tunnel_ip_range}.1/{tunnel_cidr}` → `172.16.234.1/24`
- Host 2: `{tunnel_ip_range}.2/{tunnel_cidr}` → `172.16.234.2/24`
- Host 3: `{tunnel_ip_range}.3/{tunnel_cidr}` → `172.16.234.3/24`
- And so on...

## Usage

### Deploy WireGuard Tunnels Only

```bash
ansible-playbook -i inventory/sample/hosts.yml site.yml --tags wireguard
```

### Deploy Specific Tunnel Group

```bash
ansible-playbook -i inventory/sample/hosts.yml site.yml --limit wireguard1 --tags wireguard
```

### Update Existing Tunnel Configuration

```bash
ansible-playbook -i inventory/sample/hosts.yml site.yml --tags wireguard
```

## Generated Configuration

The role creates `/etc/wireguard/wg0.conf` on each host:

```ini
[Interface]
PrivateKey = <generated_private_key>
Address = 172.16.234.1/24
ListenPort = 51820

[Peer]
PublicKey = <peer_public_key>
Endpoint = 87.247.168.200:51820
AllowedIPs = 172.16.234.0/24
PersistentKeepalive = 25
```

## Network Architecture

### Single Tunnel Example

```
┌─────────────────┐                               ┌─────────────────┐
│   machine1      │                               │    engine1      │
│                 │                               │                 │
│ Public IP:      │    WireGuard Tunnel (UDP)    │ Public IP:      │
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
│  machine1   │───── WireGuard Tunnel 1 ─────────►│   engine1   │
│ 172.16.234.1│                                   │172.16.234.2 │
└─────────────┘                                   └─────────────┘

┌─────────────┐                                   ┌─────────────┐
│  machine2   │───── WireGuard Tunnel 2 ─────────►│   engine2   │
│ 172.16.234.3│                                   │172.16.234.4 │
└─────────────┘                                   └─────────────┘

┌─────────────┐                                   ┌─────────────┐
│  machine3   │───── WireGuard Tunnel 3 ─────────►│   engine3   │
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

- Which WireGuard group the host belongs to (wireguard1, wireguard2, etc.)
- Who the peer host is (the other host in the same group)
- Peer type (src or dest)

### 2. WireGuard Setup (`tasks/wireguard.yaml`)

Performs:

- Package installation (wireguard)
- IP forwarding enablement
- Key generation (private/public key pairs)
- WireGuard configuration generation
- Network activation
- NAT/MASQUERADE setup (for dest hosts in 10.0.0.0/16)

## Verification

### Check Tunnel Status

```bash
# View WireGuard interface
sudo wg show

# Check WireGuard interface
ip addr show wg0

# View WireGuard statistics
sudo wg show wg0
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

### Issue: WireGuard Interface Not Appearing

**Symptoms**: `wg show` doesn't show the WireGuard interface

**Solutions**:

```bash
# Check WireGuard configuration
sudo cat /etc/wireguard/wg0.conf

# Check WireGuard service status
sudo systemctl status wg-quick@wg0

# Start WireGuard manually
sudo wg-quick up wg0

# Check for WireGuard errors
sudo journalctl -u wg-quick@wg0 -f
```

### Issue: Cannot Ping Peer

**Symptoms**: Tunnel exists but ping fails

**Solutions**:

```bash
# 1. Check if remote public IP is reachable
ping {peer_public_ip}

# 2. Verify WireGuard traffic is allowed
sudo tcpdump -i any port 51820

# 3. Check routing table
ip route show

# 4. Verify firewall rules
sudo iptables -L -v -n

# 5. Check if interface is UP
ip link show wg0
sudo ip link set wg0 up
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
ip addr show wg0

# Set lower MTU (accounting for WireGuard overhead)
sudo ip link set wg0 mtu 1420

# Update WireGuard config to include MTU
# Add under [Interface] section:
# MTU = 1420
```

### Issue: Firewall Blocking WireGuard

**Symptoms**: Tunnel configured but no connectivity

**Solutions**:

```bash
# Allow WireGuard traffic (UDP port 51820)
sudo iptables -A INPUT -p udp --dport 51820 -j ACCEPT
sudo iptables -A OUTPUT -p udp --sport 51820 -j ACCEPT

# For UFW users
sudo ufw allow 51820/udp

# Check if packets are being dropped
sudo iptables -L -v -n | grep DROP
```

## Security Considerations

### WireGuard Advantages

- **Strong Encryption**: Uses modern cryptography (ChaCha20, Poly1305, BLAKE2s)
- **Perfect Forward Secrecy**: Each session uses unique keys
- **Minimal Attack Surface**: Small, auditable codebase
- **No Known Vulnerabilities**: Modern, well-designed protocol

### Recommendations

1. **Key Management**: Store private keys securely (600 permissions)
2. **Firewall Rules**: Restrict WireGuard traffic to known peer IPs
3. **Regular Updates**: Keep WireGuard package updated
4. **Monitoring**: Monitor tunnel traffic for anomalies

### Firewall Example

```bash
# Only allow WireGuard from specific peer
sudo iptables -A INPUT -p udp --dport 51820 -s {peer_public_ip} -j ACCEPT
sudo iptables -A INPUT -p udp --dport 51820 -j DROP
```

## Performance Tuning

### MTU Optimization

WireGuard adds 32 bytes of overhead. Optimize MTU to avoid fragmentation:

```ini
# In WireGuard configuration
[Interface]
MTU = 1420  # 1500 - 32 bytes WireGuard overhead
```

### TCP MSS Clamping

```bash
# Clamp TCP MSS to avoid fragmentation
sudo iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```

## Advanced Configuration

### Custom Listen Port

Modify the template to change the listen port:

```ini
# In WireGuard configuration
[Interface]
ListenPort = 51821  # Default is 51820
```

### Persistent Keepalive

Configure keepalive to detect dead tunnels:

```ini
# In WireGuard configuration
[Peer]
PersistentKeepalive = 10  # Send keepalive every 10 seconds
```

### Allowed IPs Configuration

Control which IPs can be routed through the tunnel:

```ini
# In WireGuard configuration
[Peer]
AllowedIPs = 172.16.234.0/24, 10.0.0.0/8  # Multiple subnets
```

## Integration with Other Roles

This role is standalone but provides the foundation for:

- **[BIRD Role](../bird/README.md)**: Uses WireGuard tunnels for BGP peering
- **[Calico Role](../calico/README.md)**: Requires WireGuard + BIRD for multi-cluster networking

## Files Modified

- `/etc/wireguard/wg0.conf` - WireGuard tunnel configuration
- `/etc/wireguard/private_key` - Private key for this host
- `/etc/wireguard/public_key` - Public key for this host
- `/etc/sysctl.conf` - IP forwarding configuration
- `/etc/iptables/rules.v4` - NAT rules (implicitly via iptables module)

## Idempotency

This role is idempotent and can be run multiple times safely:

- WireGuard configuration is only updated if changes are detected
- Keys are only generated if they don't exist
- IP forwarding sysctl is set with `state: present`
- Package installation skips if already installed

## Related Documentation

- [Main README](../../README.md)
- [BIRD Role Documentation](../bird/README.md)
- [Calico Role Documentation](../calico/README.md)
