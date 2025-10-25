# ECMP Role

This role provides ECMP (Equal Cost Multi-Path) routing functionality for tunnel interfaces. It automatically detects VXLAN and WireGuard interfaces and sets up appropriate routing.

## Features

- **Auto-detection**: Automatically detects VXLAN and WireGuard interfaces
- **Dual tunnel support**: Handles both VXLAN and WireGuard simultaneously
- **Smart routing**: Uses kernel scope routes for single interfaces, ECMP for multiple interfaces
- **Dynamic monitoring**: Continuously monitors interface states and adjusts routing

## Requirements

- `iproute2` package
- Systemd for service management
- Root privileges for network configuration

## Variables

| Variable              | Default                          | Description                 |
| --------------------- | -------------------------------- | --------------------------- |
| `ecmp_check_interval` | `2`                              | Check interval in seconds   |
| `ecmp_script_path`    | `/usr/local/bin/ecmp-watcher.sh` | Path to ECMP watcher script |

## Usage

Include this role in your playbook:

```yaml
- name: Setup ECMP routing
  include_role:
    name: ecmp
```

## How it works

1. **VXLAN Detection**: Scans `/etc/netplan/*.yaml` files for VXLAN configurations
2. **WireGuard Detection**: Scans `/etc/wireguard/*.conf` files for WireGuard configurations
3. **Routing Logic**:
   - Single interface: Uses kernel scope route (`scope link`)
   - Multiple interfaces: Uses ECMP with equal weights
4. **Monitoring**: Continuously monitors interface states and adjusts routing

## Service

The role creates a systemd service `ecmp-watcher.service` that:

- Starts automatically on boot
- Restarts automatically if it fails
- Runs as root user
- Depends on network-online.target

## Logs

Check service logs with:

```bash
journalctl -u ecmp-watcher -f
```
