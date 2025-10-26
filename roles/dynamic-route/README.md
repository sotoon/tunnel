# Dynamic Route Role

This role deploys dynamic route monitoring for client machines that need to automatically manage routing based on probe health.

## Overview

The dynamic route role installs and configures a monitoring script that:

- Monitors probe endpoints for health status
- Automatically updates routing tables based on probe results
- Supports both single and multipath routing
- Runs every 5 seconds via systemd timer
- Removes routes when no healthy endpoints are available

## Components

### Dynamic Route Script

- Creates `/usr/local/bin/dynamic-route-parallel.sh` for automatic route management
- Performs parallel health checks on configured probe endpoints
- Updates routing tables based on probe results
- Supports both single and multipath routing scenarios

### Systemd Integration

- **Service**: `/etc/systemd/system/dynamic-route.service` - Runs the script
- **Timer**: `/etc/systemd/system/dynamic-route.timer` - Executes every 5 seconds
- Automatic restart and logging via systemd

## Usage

### Deploy to client hosts:

```yaml
- hosts: clients
  roles:
    - role: dynamic-route
```

### Manual monitoring:

```bash
# Check dynamic route status
sudo systemctl status dynamic-route.timer

# View dynamic route logs
sudo journalctl -u dynamic-route.service -f

# Run script manually
sudo /usr/local/bin/dynamic-route-parallel.sh
```

## Configuration

The dynamic route functionality is configured in `group_vars/all/tunnel.yaml`:

```yaml
# Dynamic route configuration
dynamic_route_target: "10.10.0.0/24" # Route target network
dynamic_route_interface: "eth0" # Interface to use
dynamic_route_curl_timeout: 2 # Probe timeout in seconds

# Dynamic route probes configuration
dynamic_route_probes:
  - label: "target-1"
    url: "http://10.0.0.61:9116/probe?target=192.168.234.2&module=icmp"
    next_hop: "10.0.0.61"
  - label: "target-2"
    url: "http://10.0.0.49:9116/probe?target=192.168.234.4&module=icmp"
    next_hop: "10.0.0.49"
```

## Files Created

- `/usr/local/bin/dynamic-route-parallel.sh`: Dynamic route script
- `/etc/systemd/system/dynamic-route.service`: Dynamic route service
- `/etc/systemd/system/dynamic-route.timer`: Dynamic route timer (runs every 5 seconds)

## How It Works

1. **Probe Monitoring**: The script performs parallel HTTP requests to configured probe endpoints
2. **Health Assessment**: Checks HTTP response codes and probe success metrics
3. **Route Management**:
   - Adds routes for healthy endpoints
   - Updates routes when endpoint health changes
   - Removes routes when no healthy endpoints exist
4. **Multipath Support**: Automatically configures multipath routing when multiple healthy endpoints are available

## Requirements

- Client machines must have network connectivity to probe endpoints
- Appropriate routing permissions (typically requires root/sudo)
- curl command available for HTTP probes
