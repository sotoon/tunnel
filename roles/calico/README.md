# Calico Role - Level 3: Kubernetes Multi-Cluster Connectivity

The Calico role configures BGP peering between Kubernetes clusters and tunnel endpoints (VXLAN, WireGuard, or GRE), enabling pod-to-pod and service-to-service communication across multiple geographically distributed Kubernetes clusters.

## Overview

This role is the third and final level of connectivity, building on tunnel configuration (Level 1, typically VXLAN) and BIRD BGP routing (Level 2) to connect Kubernetes clusters. It configures Calico's BGP capabilities to peer with tunnel endpoints, allowing pods in one cluster to communicate with pods in another cluster seamlessly.

**Prerequisites**: Requires both tunnel configuration (typically [VXLAN role](../vxlan/README.md)) and [BIRD role](../bird/README.md) to be configured first.

## What This Role Does

1. **Discovers Service CIDRs**: Automatically extracts service CIDR from each Kubernetes cluster
2. **Configures BGP in Calico**: Creates `BGPConfiguration` resources with:
   - Cluster AS numbers
   - Service CIDR advertisement
   - Node-to-node mesh settings
3. **Creates BGP Peers**: Establishes `BGPPeer` resources for each tunnel endpoint
4. **Enables Cross-Cluster Routing**: Routes pod and service traffic through GRE tunnels

## Requirements

- **Prerequisites**:
  - Level 1 (VXLAN/WireGuard/GRE tunnels) configured
  - Level 2 (BIRD BGP) configured
- **Kubernetes**: Two or more clusters with Calico CNI
- **Calico**: Must be installed and operational
- **Kubeconfig**: Valid kubeconfig files for all clusters
- **Ansible Collection**: `kubernetes.core`
- **Tools**: `kubectl`, `python3`

## Installation

### Install Required Ansible Collection

```bash
ansible-galaxy collection install kubernetes.core
```

### Install Python Kubernetes Client

```bash
pip3 install kubernetes
```

## Configuration

### Kubeconfig Files

In `group_vars/all/kube.yaml`:

```yaml
KUBECONFIG_1: /path/to/cluster1-kubeconfig.yaml
KUBECONFIG_2: /path/to/cluster2-kubeconfig.yaml

KUBE_WORKERS_1:
  - 10.0.0.243
  - 10.0.0.183

KUBE_WORKERS_2:
  - 10.0.10.27
  - 10.0.10.200
  - 10.0.10.136
```

### Variable Explanations

- **`KUBECONFIG_X`**: Path to kubeconfig file for cluster X
- **`KUBE_WORKERS_X`**: List of worker node IPs in cluster X (must match node IPs visible to BGP)

**Important**: Worker IPs should be the IPs that BIRD on tunnel endpoints will use for BGP peering.

## Architecture

### Multi-Cluster Network Flow

```
┌────────────────────────────────────────────────────────────┐
│              Kubernetes Cluster 1 (AS 61012)              │
│                                                            │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐              │
│  │  Pod A  │    │  Pod B  │    │Service X│              │
│  │10.244.1.│    │10.244.2.│    │10.96.0. │              │
│  └─────────┘    └─────────┘    └─────────┘              │
│       │              │               │                    │
│  ┌────▼──────────────▼───────────────▼─────┐            │
│  │    Calico CNI (Node-to-Node Mesh)       │            │
│  │    Workers: 10.0.0.243, 10.0.0.183      │            │
│  └────┬─────────────────────────────────────┘            │
└───────┼────────────────────────────────────────────────────┘
        │ BGP Peering (AS 61012 ↔ AS 61113)
        │
┌───────▼────────────────────────────────────────────────────┐
│         Tunnel Endpoint (machine1)                        │
│         AS 61113 - BIRD BGP Router                        │
│         Tunnel IP: 192.168.234.1 (VXLAN)                  │
└───────┬────────────────────────────────────────────────────┘
        │ VXLAN/WireGuard/GRE Tunnel + BGP over Tunnel
        │ (AS 61113 ↔ AS 61113)
┌───────▼────────────────────────────────────────────────────┐
│         Tunnel Endpoint (engine1)                         │
│         AS 61113 - BIRD BGP Router                        │
│         Tunnel IP: 192.168.234.2 (VXLAN)                  │
└───────┬────────────────────────────────────────────────────┘
        │ BGP Peering (AS 61113 ↔ AS 62012)
        │
┌───────▼────────────────────────────────────────────────────┐
│              Kubernetes Cluster 2 (AS 62012)              │
│                                                            │
│  ┌────────────────────────────────────────┐               │
│  │    Calico CNI (Node-to-Node Mesh)     │               │
│  │    Workers: 10.0.10.27, 10.0.10.200   │               │
│  └────┬───────────────────┬───────────────┘               │
│       │                   │                                │
│  ┌────▼────┐    ┌────▼────┐    ┌─────────┐               │
│  │  Pod C  │    │  Pod D  │    │Service Y│               │
│  │10.244.3.│    │10.244.4.│    │10.96.0. │               │
│  └─────────┘    └─────────┘    └─────────┘               │
└────────────────────────────────────────────────────────────┘
```

### AS Number Scheme

**Kubernetes Clusters**:

```
AS Number = 6{worker_index}012
```

- Cluster 1: AS 61012
- Cluster 2: AS 62012

**Tunnel Endpoints** (configured by BIRD role):

```
AS Number = 6{group_num}{workers_index}13
```

- vxlan1/wireguard1/gre1, cluster1: AS 61113
- vxlan1/wireguard1/gre1, cluster2: AS 61213
- vxlan2/wireguard2/gre2, cluster1: AS 62113
- vxlan2/wireguard2/gre2, cluster2: AS 62213

## Usage

### Deploy Calico Configuration

```bash
ansible-playbook -i inventory/sample/hosts.yml site.yml --tags calico
```

### Update BGP Peers

```bash
ansible-playbook -i inventory/sample/hosts.yml site.yml --tags calico
```

### Deploy Everything (All Three Levels)

```bash
ansible-playbook -i inventory/sample/hosts.yml site.yml
```

## Generated Kubernetes Resources

### BGPConfiguration

Created in each cluster to define BGP settings:

```yaml
apiVersion: crd.projectcalico.org/v1
kind: BGPConfiguration
metadata:
  name: default
spec:
  asNumber: 61012
  bindMode: None
  listenPort: 178
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: true
  serviceClusterIPs:
    - cidr: 10.96.0.0/12
```

### BGPPeer

Created for each tunnel endpoint in each cluster:

```yaml
apiVersion: crd.projectcalico.org/v1
kind: BGPPeer
metadata:
  name: peer-to-cluster-gre1-worker1
spec:
  asNumber: 61113
  keepOriginalNextHop: true
  nodeSelector: all()
  peerIP: "192.168.1.10"
  sourceAddress: None
```

## Task Breakdown

### 1. BGP Setup (`tasks/bgp.yaml`)

Main orchestration task:

- Gathers facts from all tunnel hosts
- Processes each worker type (cluster)
- Creates BGP peer configurations

### 2. Worker Type Processing (`tasks/process_worker_type.yaml`)

For each Kubernetes cluster:

- Discovers service CIDR from Kubernetes API
- Normalizes CIDR format
- Creates `BGPConfiguration` resource
- Validates configuration

### 3. BGP Peer Creation

For each combination of tunnel group × cluster:

- Determines correct AS number
- Identifies tunnel endpoint IP
- Creates `BGPPeer` resource

## Service CIDR Discovery

The role automatically discovers the service CIDR from each cluster:

```bash
kubectl cluster-info dump | \
  grep -m 1 service-cluster-ip-range | \
  awk -F= '{print $2}' | \
  tr -d '"'
```

Then normalizes it to proper CIDR notation using Python:

```python
import ipaddress
net = ipaddress.ip_network(cidr, strict=False)
# Normalize to base network
```

## Verification

### Check BGPConfiguration

```bash
# View BGP configuration
kubectl get bgpconfigurations -o yaml

# Check AS number and service CIDR
kubectl get bgpconfigurations default -o jsonpath='{.spec.asNumber}'
kubectl get bgpconfigurations default -o jsonpath='{.spec.serviceClusterIPs}'
```

### Check BGPPeer Resources

```bash
# List all BGP peers
kubectl get bgppeers

# Expected output:
# NAME                              AGE
# peer-to-cluster-gre1-worker1     5m
# peer-to-cluster-gre2-worker1     5m

# View peer details
kubectl get bgppeers -o yaml
```

### Check Calico Node Status

```bash
# Check BGP peering status
calicoctl node status

# Expected output showing peers:
# IPv4 BGP status
# +---------------+-------------+----------+----------+
# |  PEER ADDRESS |    PEER     |  STATE   | UPTIME   |
# +---------------+-------------+----------+----------+
# | 192.168.1.10  | AS(61113)   | up       | 10m15s   |
# | 10.0.0.183    | node-mesh   | up       | 2d       |
# +---------------+-------------+----------+----------+
```

### Check Routes in Calico

```bash
# View routes learned via BGP
calicoctl get bgp routes

# Check specific node routes
kubectl exec -n kube-system calico-node-xxx -- ip route show
```

### Test Cross-Cluster Connectivity

#### Pod-to-Pod

```bash
# Create test pod in cluster 1
kubectl --context cluster1 run test-pod --image=busybox --restart=Never -- sleep 3600

# Get pod IP
POD_IP=$(kubectl --context cluster1 get pod test-pod -o jsonpath='{.status.podIP}')

# From cluster 2, try to ping cluster 1 pod
kubectl --context cluster2 exec -it another-pod -- ping $POD_IP
```

#### Service Access

```bash
# Create service in cluster 1
kubectl --context cluster1 create deployment nginx --image=nginx
kubectl --context cluster1 expose deployment nginx --port=80

# Get service IP
SVC_IP=$(kubectl --context cluster1 get svc nginx -o jsonpath='{.spec.clusterIP}')

# From cluster 2, access the service
kubectl --context cluster2 exec -it test-pod -- wget -O- http://$SVC_IP
```

## Troubleshooting

### Issue: BGP Peers Not Establishing

**Symptoms**: `calicoctl node status` shows peers in "down" or "idle" state

**Solutions**:

```bash
# 1. Check if tunnel endpoints are reachable
kubectl exec -n kube-system calico-node-xxx -- ping {tunnel_endpoint_ip}

# 2. Check Calico logs
kubectl logs -n kube-system -l k8s-app=calico-node --tail=100 | grep -i bgp

# 3. Verify BGPPeer configuration
kubectl get bgppeers -o yaml

# 4. Check if port 178 is accessible
kubectl exec -n kube-system calico-node-xxx -- telnet {tunnel_endpoint_ip} 178

# 5. Verify AS numbers match
# BGPPeer AS should match BIRD AS on tunnel endpoint
kubectl get bgppeers -o jsonpath='{.items[*].spec.asNumber}'
sudo birdc show protocols all | grep "AS"  # On tunnel endpoint
```

### Issue: Routes Not Propagating

**Symptoms**: BGP peers are up but routes are missing

**Solutions**:

```bash
# 1. Check what routes Calico is advertising
calicoctl get bgp routes

# 2. Check BIRD on tunnel endpoint
sudo birdc show route protocol bgp_to_worker_xxx

# 3. Verify service CIDR is correct
kubectl get bgpconfigurations default -o jsonpath='{.spec.serviceClusterIPs}'

# 4. Check route filters in BIRD
# On tunnel endpoint
sudo birdc show route export bgp_to_worker_xxx

# 5. Verify IPPool advertisements
kubectl get ippools -o yaml
# Ensure natOutgoing is configured correctly
```

### Issue: Service CIDR Discovery Fails

**Symptoms**: Error during playbook run about service CIDR

**Solutions**:

```bash
# 1. Manually check service CIDR
kubectl cluster-info dump | grep service-cluster-ip-range

# 2. Set manually in BGPConfiguration if needed
kubectl apply -f - <<EOF
apiVersion: crd.projectcalico.org/v1
kind: BGPConfiguration
metadata:
  name: default
spec:
  serviceClusterIPs:
    - cidr: 10.96.0.0/12  # Your service CIDR
EOF

# 3. Check Python3 is available
python3 --version

# 4. Test CIDR normalization manually
python3 -c "import ipaddress; print(ipaddress.ip_network('10.96.0.10/12', strict=False))"
```

### Issue: Cross-Cluster Connectivity Fails

**Symptoms**: Cannot ping/access pods across clusters

**Solutions**:

```bash
# 1. Verify BGP is working at all levels
# On Calico node
calicoctl node status

# On tunnel endpoint
sudo birdc show protocols

# 2. Check routing table
kubectl exec -n kube-system calico-node-xxx -- ip route show | grep {remote_subnet}

# 3. Verify IP-in-IP or VXLAN is not interfering
kubectl get ippools -o yaml
# Check: ipipMode: Never, vxlanMode: Never (for BGP routing)

# 4. Check firewall rules on tunnel endpoints
sudo iptables -L -v -n

# 5. Trace packet path
kubectl exec -n kube-system calico-node-xxx -- traceroute {remote_pod_ip}

# 6. Check Calico Felix logs
kubectl logs -n kube-system -l k8s-app=calico-node -c calico-felix
```

### Issue: "keepOriginalNextHop" Warning

**Symptoms**: Warnings about next hop in logs

**Explanation**: `keepOriginalNextHop: true` tells Calico to preserve the original next hop, which may cause warnings if the next hop is not directly reachable.

**Solutions**:

```bash
# If you see next hop issues, try setting to false
kubectl patch bgppeers peer-to-cluster-gre1-worker1 \
  --type=merge -p '{"spec":{"keepOriginalNextHop":false}}'
```

### Issue: Node Mesh Conflicts

**Symptoms**: Routes are flapping or incorrect

**Solutions**:

```bash
# Check if node mesh is enabled
kubectl get bgpconfigurations default -o jsonpath='{.spec.nodeToNodeMeshEnabled}'

# For multi-cluster setups, node mesh should typically be enabled
# within each cluster but not across clusters

# Verify only external peers are tunnel endpoints
kubectl get bgppeers -o custom-columns=NAME:.metadata.name,IP:.spec.peerIP
```

## Security Considerations

### BGP Security

1. **Peer Authentication**: Consider adding BGP MD5 passwords
2. **Route Filtering**: Calico accepts all routes - consider adding filters
3. **Network Policies**: Use Calico NetworkPolicies to restrict cross-cluster traffic

### Example: Restrict Cross-Cluster Traffic

```yaml
apiVersion: crd.projectcalico.org/v1
kind: GlobalNetworkPolicy
metadata:
  name: allow-cross-cluster
spec:
  order: 100
  selector: app == 'allowed-app'
  types:
    - Ingress
  ingress:
    - action: Allow
      source:
        nets:
          - 10.244.0.0/16 # Remote cluster pod CIDR
```

### Firewall Recommendations

```bash
# On Kubernetes nodes, allow BGP from tunnel endpoints only
iptables -A INPUT -p tcp --dport 178 -s {tunnel_endpoint_ip} -j ACCEPT
iptables -A INPUT -p tcp --dport 178 -j DROP
```

## Performance Tuning

### BGP Timers

Calico uses default BGP timers. These cannot be configured per-peer but can be adjusted globally.

### Route Reflectors

For large clusters (>100 nodes), consider using Calico route reflectors:

```yaml
apiVersion: crd.projectcalico.org/v1
kind: BGPConfiguration
metadata:
  name: default
spec:
  nodeToNodeMeshEnabled: false # Disable full mesh

---
apiVersion: crd.projectcalico.org/v1
kind: BGPPeer
metadata:
  name: route-reflector
spec:
  nodeSelector: route-reflector == 'true'
  peerSelector: all()
```

### Optimize Service Advertisement

Only advertise service CIDR if you need cross-cluster service access:

```yaml
# To disable service advertisement
apiVersion: crd.projectcalico.org/v1
kind: BGPConfiguration
metadata:
  name: default
spec:
  serviceClusterIPs: [] # Empty list = don't advertise
```

## Advanced Configuration

### Multiple Clusters (>2)

To support more than 2 clusters, add additional KUBECONFIG entries:

```yaml
KUBECONFIG_3: /path/to/cluster3-kubeconfig.yaml
KUBE_WORKERS_3:
  - 10.0.20.10
  - 10.0.20.11
```

Update the role to process `worker_types` dynamically.

### Selective Peering

To peer only specific nodes with external BGP:

```yaml
apiVersion: crd.projectcalico.org/v1
kind: BGPPeer
metadata:
  name: peer-to-tunnel
spec:
  nodeSelector: bgp-peer == 'external' # Only nodes with this label
  peerIP: "192.168.1.10"
  asNumber: 61113
```

Then label specific nodes:

```bash
kubectl label nodes worker1 bgp-peer=external
```

## Integration with Other Roles

- **[VXLAN Role](../vxlan/README.md)**: Primary tunnel type - provides VXLAN tunnel infrastructure
- **[WireGuard Role](../wireguard/README.md)**: Alternative tunnel type - provides encrypted tunnels
- **[GRE Role](../gre/README.md)**: Alternative tunnel type - provides GRE tunnels
- **[BIRD Role](../bird/README.md)**: Required - provides BGP routing between tunnels

## Files Modified

None on the target hosts. All changes are applied to Kubernetes clusters via API.

## Kubernetes Resources Created

- `BGPConfiguration`: One per cluster
- `BGPPeer`: Multiple per cluster (one per tunnel endpoint × tunnel group)

## Idempotency

This role is idempotent:

- Kubernetes resources are created with `state: present` (updates if exists)
- Service CIDR discovery runs each time (safe, read-only operation)
- No changes are made if configuration matches existing state

## Related Documentation

- [Main README](../../README.md)
- [VXLAN Role Documentation](../vxlan/README.md) - Level 1 primary prerequisite
- [BIRD Role Documentation](../bird/README.md) - Level 2 prerequisite
- [Calico BGP Documentation](https://docs.tigera.io/calico/latest/networking/configuring/bgp)
- [Project Calico](https://www.projectcalico.org/)
