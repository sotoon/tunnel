# BIRD Role - Level 2: BGP Route Propagation

The BIRD role configures BIRD2 (BIRD Internet Routing Daemon) to propagate routes between machines using BGP (Border Gateway Protocol). This enables dynamic routing, automatic failover, and load balancing across multiple GRE tunnels.

## Overview

This role builds on Level 1 (GRE tunnels) to add intelligent route propagation. It's useful when you have multiple paths between sites and want automatic route selection based on priority, load balancing, or failover scenarios.

**Prerequisites**: Requires [GRE role](../gre/README.md) to be configured first.

## What This Role Does

1. **Installs BIRD2**: Installs the BIRD2 BGP daemon
2. **Configures BGP**: Generates BIRD configuration with:
   - BGP sessions between tunnel endpoints
   - BGP sessions with Kubernetes worker nodes
   - Route filtering and manipulation
   - Priority-based path selection
3. **Manages Service**: Restarts BIRD2 when configuration changes

## Requirements

- **Prerequisites**: Level 1 (GRE tunnels) must be configured
- **Operating System**: Ubuntu (uses apt)
- **Network**: BGP port access (TCP 179)
- **Privileges**: Root/sudo access

## Configuration

### Inventory Variables

The same inventory used for GRE role is used here. BIRD automatically discovers:

- Tunnel peers from GRE groups
- Local and remote tunnel IPs
- Network subnets to advertise

### Global Variables

In `group_vars/all/kube.yaml`:

```yaml
KUBE_WORKERS_1:
  - 10.0.0.243
  - 10.0.0.183
KUBE_WORKERS_2:
  - 10.0.10.27
  - 10.0.10.200
```

These define Kubernetes worker nodes for BGP peering (can be empty if not using Kubernetes).

## BGP Architecture

### AS Number Scheme

BIRD uses a dynamic AS number scheme based on tunnel group and worker index:

**For Tunnel Endpoints**:

```
AS Number = 6{gre_index}{worker_index}13
```

Examples:

- gre1, worker1: AS 61113
- gre1, worker2: AS 61213
- gre2, worker1: AS 62113
- gre2, worker2: AS 62213

**For Kubernetes Workers**:

```
AS Number = 6{worker_index}012
```

Examples:

- Cluster 1 workers: AS 61012
- Cluster 2 workers: AS 62012

### BGP Sessions

Each tunnel endpoint establishes:

1. **Peer-to-Peer Session**: BGP with the other tunnel endpoint
2. **Worker Sessions**: BGP with each Kubernetes worker node (if configured)

### Network Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster 1                    │
│  Worker 1 (10.0.0.243)    Worker 2 (10.0.0.183)            │
│       AS 61012                  AS 61012                    │
└───────┬──────────────────────────┬──────────────────────────┘
        │ BGP                      │ BGP
        │                          │
┌───────▼──────────────────────────▼──────────────────────────┐
│  machine1 (tunnel endpoint)                                 │
│  AS 61113                                                   │
│  Tunnel IP: 172.16.234.1                                    │
└───────┬─────────────────────────────────────────────────────┘
        │ BGP over GRE Tunnel
        │
┌───────▼─────────────────────────────────────────────────────┐
│  engine1 (tunnel endpoint)                                  │
│  AS 61113                                                   │
│  Tunnel IP: 172.16.234.2                                    │
└───────┬──────────────────────────┬──────────────────────────┘
        │ BGP                      │ BGP
        │                          │
┌───────▼──────────────────────────▼──────────────────────────┐
│                     Kubernetes Cluster 2                    │
│  Worker 1 (10.0.10.27)    Worker 2 (10.0.10.200)           │
│       AS 62012                  AS 62012                    │
└─────────────────────────────────────────────────────────────┘
```

## Route Priority and Manipulation

BIRD implements sophisticated route selection using multiple BGP attributes:

### 1. Local Preference (Higher = Better)

```
Local Preference = (3 - gre_index + 1) * 100
```

- gre1: 300
- gre2: 200
- gre3: 100

**Impact**: Routes through gre1 are most preferred, gre3 least preferred.

### 2. AS Path Prepending (Shorter = Better)

```
Prepend Count = gre_index - 1
```

- gre1: No prepending (shortest path)
- gre2: Prepend AS once (path length +1)
- gre3: Prepend AS twice (path length +2)

**Impact**: Makes paths through higher-indexed tunnels appear longer, reducing their preference.

### 3. MED - Multi-Exit Discriminator (Lower = Better)

```
MED = gre_index * 100
```

- gre1: 100
- gre2: 200
- gre3: 300

**Impact**: gre1 has lowest MED (most preferred), gre3 has highest (least preferred).

### Combined Effect

The three mechanisms work together to create a clear priority:

**gre1 > gre2 > gre3**

This ensures:

- Primary traffic flows through gre1
- gre2 is used as backup or for load balancing
- gre3 is last resort

## Usage

### Deploy BIRD on All GRE Endpoints

```bash
ansible-playbook -i inventory/sample/hosts.yml site.yml --tags bird
```

### Deploy BIRD on Specific Tunnel

```bash
ansible-playbook -i inventory/sample/hosts.yml site.yml --limit gre1 --tags bird
```

### Update BIRD Configuration

```bash
ansible-playbook -i inventory/sample/hosts.yml site.yml --tags bird
```

The BIRD service will restart automatically if configuration changes.

## Generated Configuration

BIRD configuration is generated at `/etc/bird/bird.conf`:

### Key Sections

#### 1. Router ID

```
router id 172.16.234.1;
```

Set to the tunnel local IP address.

#### 2. Static Routes

```
protocol static {
    ipv4;
    route 192.168.1.0/24 via "eth0";           # Local subnet
    route 172.16.234.1/32 via "gre";            # Tunnel endpoint IP
    route 172.16.234.2/32 via "gre";            # Peer endpoint IP
}
```

#### 3. BGP to Peer Tunnel Endpoint

```
protocol bgp bgp_to_node2 {
    local 172.16.234.1 as 61113;
    neighbor 172.16.234.2 as 61113;
    ipv4 {
        export filter nexthops;
        import all;
        next hop self;
    };
    direct;
    hold time 240;
    keepalive time 80;
}
```

#### 4. BGP to Kubernetes Workers

```
protocol bgp bgp_to_worker_243 {
    local 192.168.1.10 as 61113;
    neighbor 10.0.0.243 as 61012;
    ipv4 {
        export filter nexthops;
        import all;
        next hop self;
    };
    direct;
}
```

#### 5. Route Filter

```
function set_nexthops()
{
    bgp_med = 100;
    bgp_local_pref = 300;
    # AS path prepending (if gre_index > 1)
    accept;
}

filter nexthops {
    set_nexthops();
}
```

## Verification

### Check BGP Status

```bash
# View all BGP protocols
sudo birdc show protocols

# Expected output:
# bgp_to_node2           BGP      up     <timestamp>  Established
# bgp_to_worker_243      BGP      up     <timestamp>  Established
```

### View BGP Routes

```bash
# Show all routes
sudo birdc show route

# Show routes from specific protocol
sudo birdc show route protocol bgp_to_node2

# Show routes being exported
sudo birdc show route export bgp_to_node2
```

### Check BGP Session Details

```bash
# Show detailed BGP information
sudo birdc show protocols all bgp_to_node2

# Shows:
# - BGP state
# - Uptime
# - Route import/export counts
# - Hold time, keepalive
```

### View BGP Attributes

```bash
# Show routes with all BGP attributes
sudo birdc show route all

# Look for:
# - BGP.local_pref
# - BGP.med
# - BGP.as_path
```

## Troubleshooting

### Issue: BGP Session Not Establishing

**Symptoms**: `show protocols` shows state other than "Established"

**Solutions**:

```bash
# 1. Check BIRD is running
sudo systemctl status bird

# 2. Check BIRD configuration syntax
sudo bird -p -c /etc/bird/bird.conf

# 3. View BIRD logs
sudo journalctl -u bird -f

# 4. Check network connectivity to peer
ping {peer_tunnel_ip}
telnet {peer_tunnel_ip} 179

# 5. Verify firewall allows BGP
sudo iptables -L -v -n | grep 179

# 6. Check AS numbers match expectations
sudo birdc show protocols all bgp_to_node2 | grep "AS"
```

### Issue: Routes Not Being Advertised

**Symptoms**: BGP is up but routes are missing

**Solutions**:

```bash
# 1. Check what routes are being exported
sudo birdc show route export bgp_to_node2

# 2. Check route filters
sudo birdc eval "show route filter nexthops"

# 3. Check static routes are configured
sudo birdc show route protocol static

# 4. Verify kernel routes exist
ip route show

# 5. Check if next hop self is working
sudo birdc show route all | grep "next hop"
```

### Issue: Routes Present but Not Preferred

**Symptoms**: Routes exist but traffic uses wrong path

**Solutions**:

```bash
# 1. Check BGP attributes
sudo birdc show route all | grep -A 5 "192.168"

# 2. Verify local preference
# Should show: BGP.local_pref: 300 (gre1), 200 (gre2), etc.

# 3. Check MED values
# Should show: BGP.med: 100 (gre1), 200 (gre2), etc.

# 4. Check AS path length
# Should show: BGP.as_path: [61113] (gre1), [61113 61113] (gre2), etc.

# 5. View best path selection
sudo birdc show route primary
```

### Issue: BIRD Service Fails to Start

**Symptoms**: `systemctl start bird` fails

**Solutions**:

```bash
# 1. Test configuration syntax
sudo bird -p -c /etc/bird/bird.conf

# 2. View detailed error
sudo journalctl -u bird -n 50

# 3. Check file permissions
ls -l /etc/bird/bird.conf
# Should be readable by bird user

# 4. Check for port conflicts
sudo netstat -tlnp | grep 179

# 5. Start in foreground for debugging
sudo bird -f -c /etc/bird/bird.conf -d
```

### Issue: High CPU Usage

**Symptoms**: BIRD consuming excessive CPU

**Solutions**:

```bash
# 1. Check for route flapping
sudo birdc show protocols all | grep "Import updates"

# 2. Increase timers to reduce overhead
# In bird.conf:
#   scan time 120;  # Instead of 60

# 3. Check for excessive logging
# In bird.conf:
#   log syslog { warning, error };  # Instead of 'all'
```

## BGP Timers

### Default Timers

```
hold time 240        # Session considered dead after 240s
keepalive time 80    # Send keepalive every 80s
connect delay 2      # Wait 2s before retry
connect retry 5      # Retry every 5s if connection fails
error wait 5,30      # Wait 5-30s after errors
```

### Tuning Recommendations

**Fast Convergence** (for critical paths):

```
hold time 90
keepalive time 30
```

**Slow Networks** (reduce overhead):

```
hold time 600
keepalive time 200
```

## Route Filtering

### Current Filter

The `nexthops` filter applies to all exported routes:

- Sets BGP MED based on GRE index
- Sets BGP local preference
- Prepends AS path for priority
- Accepts all routes

### Custom Filters Example

To advertise only specific networks:

```
filter nexthops {
    if net ~ [ 192.168.0.0/16+ ] then {
        set_nexthops();
    } else {
        reject;
    }
}
```

To block specific routes:

```
filter nexthops {
    if net = 10.0.0.0/8 then reject;
    set_nexthops();
}
```

## Security Considerations

### BGP Security Risks

- **Route Hijacking**: Malicious peers can advertise incorrect routes
- **No Authentication**: Default configuration lacks BGP MD5 auth
- **DoS Attacks**: BGP floods can overwhelm the daemon

### Recommendations

#### 1. BGP MD5 Authentication

```
protocol bgp bgp_to_node2 {
    password "your-secret-key";
    # ... rest of config
}
```

#### 2. Maximum Prefix Limits

```
protocol bgp bgp_to_node2 {
    ipv4 {
        import limit 1000;  # Reject if peer sends >1000 routes
    };
}
```

#### 3. Firewall Rules

```bash
# Only allow BGP from known peers
sudo iptables -A INPUT -p tcp --dport 179 -s {peer_ip} -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 179 -j DROP
```

#### 4. Route Filtering

Always filter routes to only advertise what you own:

```
filter nexthops {
    if net ~ [ 192.168.1.0/24 ] then {  # Only your networks
        set_nexthops();
    } else {
        reject;
    }
}
```

## Performance Monitoring

### Monitor BGP Sessions

```bash
# Create monitoring script
cat > /usr/local/bin/bird-monitor.sh << 'EOF'
#!/bin/bash
while true; do
    echo "=== $(date) ==="
    sudo birdc show protocols | grep bgp
    sleep 60
done
EOF

chmod +x /usr/local/bin/bird-monitor.sh
```

### Log BGP Changes

```bash
# View BGP state changes
sudo journalctl -u bird -f | grep -i "bgp"
```

### Route Count Tracking

```bash
# Count routes by protocol
sudo birdc show route count
```

## Integration with Other Roles

- **[GRE Role](../gre/README.md)**: Required prerequisite - provides tunnel infrastructure
- **[Calico Role](../calico/README.md)**: Optional next step - connects Kubernetes clusters

## Files Modified

- `/etc/bird/bird.conf` - BIRD2 configuration
- `/etc/systemd/system/bird.service` - Service file (managed by package)

## Handlers

The role includes a handler that restarts BIRD2 when configuration changes:

```yaml
- name: Restart bird2
  service:
    name: bird
    state: restarted
```

## Idempotency

This role is idempotent:

- BIRD configuration is only updated if changes are detected
- Service only restarts when config changes (via handler)
- Package installation skips if already installed

## Related Documentation

- [Main README](../../README.md)
- [GRE Role Documentation](../gre/README.md) - Required prerequisite
- [Calico Role Documentation](../calico/README.md) - Next level
- [BIRD Documentation](https://bird.network.cz/)
