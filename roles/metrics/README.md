# Metrics Role

This role deploys blackbox monitoring for tunnel connectivity on tunnel machines.

## Overview

The metrics role installs and configures Prometheus Blackbox Exporter to monitor tunnel connectivity by pinging specific IP addresses based on the host type:

- **Machine hosts**: Ping destination tunnel IPs (192.168.234.2, 192.168.234.4)
- **Engine hosts**: Ping source tunnel IPs (192.168.234.1, 192.168.234.3)

## Components

### Blackbox Exporter

- Installs prometheus-blackbox-exporter package
- Configures ICMP monitoring modules
- Runs on port 9115
- Provides metrics endpoint for Prometheus scraping

### Monitoring Script

- Creates `/usr/local/bin/tunnel-monitor.sh` for manual connectivity testing
- Continuously pings target IPs with 30-second intervals
- Provides real-time connectivity status

### Prometheus Configuration

- Generates `/etc/blackbox_exporter/prometheus-targets.yml`
- Contains scrape configuration for Prometheus
- Includes relabeling for proper metric labeling

## Usage

### Deploy to tunnel machines:

```yaml
- hosts: machine
  roles:
    - role: metrics

- hosts: engine
  roles:
    - role: metrics
```

### Manual monitoring:

```bash
# Run the monitoring script
sudo /usr/local/bin/tunnel-monitor.sh

# Check blackbox exporter status
sudo systemctl status blackbox-exporter

# View metrics
curl http://localhost:9115/metrics

```

## Prometheus Integration

Include the generated targets configuration in your Prometheus config:

```yaml
# In prometheus.yml
- job_name: "tunnel-connectivity"
  file_sd_configs:
    - files:
        - "/etc/blackbox_exporter/prometheus-targets.yml"
```

## Metrics

The blackbox exporter provides the following metrics:

- `probe_success`: 1 if target is reachable, 0 if not
- `probe_duration_seconds`: Time taken for the probe
- `probe_icmp_duration_seconds`: ICMP round-trip time

## Files Created

- `/etc/blackbox_exporter/blackbox.yml`: Blackbox exporter configuration
- `/etc/systemd/system/blackbox-exporter.service`: Systemd service
- `/usr/local/bin/tunnel-monitor.sh`: Monitoring script
- `/etc/blackbox_exporter/prometheus-targets.yml`: Prometheus targets
