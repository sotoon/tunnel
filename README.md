# WireGuard Tunnel Configuration

This Ansible playbook sets up WireGuard tunnels with support for multiple WireGuard interfaces per physical network interface.

## Features

- **Dynamic Interface Configuration**: Configure different physical interfaces per host
- **Multiple WireGuard per Interface**: Support for multiple WireGuard tunnels per physical interface
- **Flexible Interface Names**: Support for any interface naming convention (eth, ens, etc.)
- **Automatic IP Range Management**: Each physical interface gets its own IP range
- **Multipath Routing**: Load balancing across multiple WireGuard tunnels

## Configuration

### Inventory Configuration

Each host can define its physical interfaces and the number of WireGuard tunnels per interface:

```yaml
all:
  hosts:
    machine1:
      ansible_host: your-server-ip
      tunnel_type: src
      public_ip: your-public-ip
      physical_interfaces:
        - name: eth0
          wg_count: 4
        - name: eth1
          wg_count: 4
    engine1:
      ansible_host: peer-server-ip
      tunnel_type: dest
      public_ip: peer-public-ip
      physical_interfaces:
        - name: eth0
          wg_count: 4
        - name: eth1
          wg_count: 4
```

### Interface Naming

The system supports any interface naming convention:

- `eth0`, `eth1` (traditional)
- `ens18`, `ens19` (systemd predictable naming)
- `enp0s3`, `enp0s8` (PCI-based naming)
- Any other interface name

### IP Range Configuration

Each physical interface gets its own IP range to avoid conflicts:

```yaml
interface_base_ranges:
  eth0:
    machine_range: 10.0.0.0/24
    engine_range: 10.10.0.0/24
  eth1:
    machine_range: 10.1.0.0/24
    engine_range: 10.11.0.0/24
```

## WireGuard Interface Generation

The system automatically generates WireGuard interfaces based on the configuration:

- **Interface Naming**: `wg0`, `wg1`, `wg2`, etc.
- **Port Assignment**: Starting from `wireguard_port` (default: 51820)
- **IP Assignment**: Each interface gets a unique IP in the tunnel range

### Example with 2 physical interfaces, 4 WireGuard each:

| Physical Interface | WireGuard Interface | Port  | IP Range         |
| ------------------ | ------------------- | ----- | ---------------- |
| eth0               | wg0                 | 51820 | 172.16.234.1/24  |
| eth0               | wg1                 | 51821 | 172.16.234.11/24 |
| eth0               | wg2                 | 51822 | 172.16.234.21/24 |
| eth0               | wg3                 | 51823 | 172.16.234.31/24 |
| eth1               | wg4                 | 51824 | 172.16.234.41/24 |
| eth1               | wg5                 | 51825 | 172.16.234.51/24 |
| eth1               | wg6                 | 51826 | 172.16.234.61/24 |
| eth1               | wg7                 | 51827 | 172.16.234.71/24 |

## Usage

1. **Configure your inventory** in `inventory/your-environment/hosts.yml`
2. **Update interface ranges** in `group_vars/all/tunnel.yaml` if needed
3. **Run the playbook**:

```bash
ansible-playbook -i inventory/your-environment/hosts.yml site.yml
```

## Advanced Configuration

### Custom Interface Ranges

You can define custom IP ranges for different physical interfaces:

```yaml
interface_base_ranges:
  eth0:
    machine_range: 10.0.0.0/24
    engine_range: 10.10.0.0/24
  ens18:
    machine_range: 10.2.0.0/24
    engine_range: 10.12.0.0/24
```

### Different WireGuard Counts

You can have different numbers of WireGuard tunnels per interface:

```yaml
physical_interfaces:
  - name: eth0
    wg_count: 2
  - name: eth1
    wg_count: 6
```

This will create:

- 2 WireGuard interfaces on eth0 (wg0, wg1)
- 6 WireGuard interfaces on eth1 (wg2, wg3, wg4, wg5, wg6, wg7)

## Troubleshooting

### Check WireGuard Status

```bash
# List all WireGuard interfaces
wg show

# Check interface status
ip a show wg0
ip a show wg1
# ... etc
```

### Check Routing

```bash
# Check multipath routes
ip route show | grep 172.16.234

# Check specific interface routes
ip route show dev wg0
```

### Logs

```bash
# Check WireGuard service status
systemctl status wg-quick@wg0
journalctl -u wg-quick@wg0
```

## Security Notes

- Private keys are stored with 600 permissions
- Public keys are stored with 644 permissions
- WireGuard configs are stored with 600 permissions
- All keys are generated per-interface for security isolation
