# Kubernetes From-Ground-Up: Pod Network Troubleshooting Runbook

## The Problem

Cross-node pod-to-pod communication fails with `Connection timed out`  
even when all Kubernetes components are healthy.

**Symptoms:**

- `kubectl exec -it <pod> -- curl <pod-ip>` times out
- `kubectl logs` and `kubectl exec` fail with `i/o timeout`
- Pods on the same node can communicate
- Pods on different nodes cannot communicate

---

## Root Cause (The One That Actually Mattered)

**Missing security group self-referencing rule.**

AWS security groups operate at the **hypervisor level** — before packets
reach the instance. Without a self-referencing rule, AWS silently drops
all traffic between instances in the cluster, regardless of:

- iptables rules
- Kernel routing tables
- Source/destination check settings
- Network ACLs

### The Fix

```bash
SECURITY_GROUP_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=k8s-cluster-from-ground-up" \
  --query 'SecurityGroups[0].GroupId' --output text)

# Allow ALL traffic between instances in the same security group
aws ec2 authorize-security-group-ingress \
  --group-id $SECURITY_GROUP_ID \
  --protocol -1 \
  --source-group $SECURITY_GROUP_ID
```

### Why This Works

The self-referencing rule tells AWS:

> "Allow all traffic from any instance that belongs to this security group"

This covers:

- Node-to-node communication (ICMP, arbitrary TCP/UDP)
- Pod-to-pod traffic across nodes
- kubelet API calls (port 10250) between master and workers

---

## Complete Security Group Rules Required

| Port/Protocol   | Source         | Purpose                   |
| --------------- | -------------- | ------------------------- |
| ALL traffic     | Self (same SG) | Node-to-node + pod-to-pod |
| TCP 22          | 0.0.0.0/0      | SSH access                |
| TCP 6443        | 0.0.0.0/0      | Kubernetes API server     |
| TCP 2379-2380   | 172.31.0.0/24  | etcd peer + client        |
| TCP 10250       | 172.31.0.0/24  | kubelet API               |
| TCP 30000-32767 | 0.0.0.0/0      | NodePort services         |
| ICMP -1         | 0.0.0.0/0      | Ping/diagnostics          |

---

## Other Required Configurations (Don't Skip These)

### 1. Disable Source/Destination Check on ALL worker nodes

AWS drops packets where the source or destination IP is not registered
to the sending instance. Pod IPs are not registered — so this must be off.

```bash
for IP in 172.31.0.20 172.31.0.21 172.31.0.22; do
  INSTANCE_ID=$(aws ec2 describe-instances \
    --filters "Name=private-ip-address,Values=${IP}" \
    --query 'Reservations[0].Instances[0].InstanceId' --output text)
  aws ec2 modify-instance-attribute \
    --instance-id $INSTANCE_ID \
    --no-source-dest-check
  echo "Disabled src/dst check on $INSTANCE_ID ($IP)"
done
```

### 2. Add Pod CIDR Routes to the Route Table

The VPC route table must know which node owns which pod CIDR.

```bash
ROUTE_TABLE_ID=rtb-xxxxxxxxxxxxxxxxx  # your route table ID

aws ec2 create-route \
  --route-table-id $ROUTE_TABLE_ID \
  --destination-cidr-block 172.20.0.0/24 \
  --instance-id <worker-0-instance-id>

aws ec2 create-route \
  --route-table-id $ROUTE_TABLE_ID \
  --destination-cidr-block 172.20.1.0/24 \
  --instance-id <worker-1-instance-id>

aws ec2 create-route \
  --route-table-id $ROUTE_TABLE_ID \
  --destination-cidr-block 172.20.2.0/24 \
  --instance-id <worker-2-instance-id>
```

### 3. Add Pod CIDR Routes on Each Worker Node

The OS routing table on each worker must know how to reach other workers'
pod CIDRs. These do not persist across reboots without the startup script.

```bash
# On worker-0 (172.31.0.20)
sudo ip route add 172.20.1.0/24 via 172.31.0.21 dev eth0
sudo ip route add 172.20.2.0/24 via 172.31.0.22 dev eth0

# On worker-1 (172.31.0.21)
sudo ip route add 172.20.0.0/24 via 172.31.0.20 dev eth0
sudo ip route add 172.20.2.0/24 via 172.31.0.22 dev eth0

# On worker-2 (172.31.0.22)
sudo ip route add 172.20.0.0/24 via 172.31.0.20 dev eth0
sudo ip route add 172.20.1.0/24 via 172.31.0.21 dev eth0
```

**Make persistent (run on each worker):**

```bash
# Replace the IPs below to match each worker's peers
cat <<EOF | sudo tee /etc/networkd-dispatcher/routable.d/k8s-pod-routes.sh
#!/bin/bash
ip route add 172.20.1.0/24 via 172.31.0.21 dev eth0
ip route add 172.20.2.0/24 via 172.31.0.22 dev eth0
EOF
sudo chmod +x /etc/networkd-dispatcher/routable.d/k8s-pod-routes.sh
```

### 4. iptables FORWARD Rules for Pod CIDR

Allow forwarding of pod network traffic on each worker node:

```bash
sudo iptables -I FORWARD 1 -s 172.20.0.0/16 -j ACCEPT
sudo iptables -I FORWARD 2 -d 172.20.0.0/16 -j ACCEPT
sudo iptables -t nat -A POSTROUTING -s 172.20.0.0/16 ! -d 172.20.0.0/16 -j MASQUERADE

# Make persistent
sudo apt-get install -y iptables-persistent
sudo netfilter-persistent save
```

---

## Diagnostic Checklist

When pod-to-pod communication fails, work through this list in order:

### Step 1 – Check security group self-referencing rule

```bash
aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=k8s-cluster-from-ground-up" \
  --query 'SecurityGroups[0].IpPermissions'
# Look for a rule with UserIdGroupPairs referencing the same group
```

### Step 2 – Check node-to-node connectivity first

```bash
# If node-to-node ping fails, the problem is AWS-level (SG or routes)
# not Kubernetes-level
ping -c 3 <other-worker-node-ip>
```

### Step 3 – Check source/destination check

```bash
aws ec2 describe-instances \
  --instance-ids <id1> <id2> <id3> \
  --query 'Reservations[*].Instances[*].{ID:InstanceId,SrcDstCheck:SourceDestCheck}' \
  --output table
# All workers must show False
```

### Step 4 – Check VPC route table

```bash
aws ec2 describe-route-tables \
  --route-table-ids <route-table-id> \
  --query 'RouteTables[0].Routes[*].{Dest:DestinationCidrBlock,Instance:InstanceId,State:State}' \
  --output table
# Must have entries for 172.20.0.0/24, 172.20.1.0/24, 172.20.2.0/24
```

### Step 5 – Check OS routing table on the source worker

```bash
ip route show
# Must have routes to other workers' pod CIDRs
# e.g. 172.20.2.0/24 via 172.31.0.22 dev eth0
```

### Step 6 – Use tcpdump to find where packets die

```bash
# On SOURCE worker - do packets leave?
sudo tcpdump -i eth0 host <destination-pod-ip> -n

# On DESTINATION worker - do packets arrive?
sudo tcpdump -i eth0 src host <source-node-ip> -n
sudo tcpdump -i eth0 -n src net 172.20.0.0/16

# If packets leave source but never arrive at destination = AWS dropping them
# If packets arrive at destination node but don't reach pod = iptables/bridge issue
```

### Step 7 – Check kubelet port (for kubectl exec/logs issues)

```bash
# On worker node
sudo ss -tlnp | grep 10250
sudo systemctl status kubelet

# From master
nc -zv <worker-ip> 10250
```

---

## Quick Reference: Verification Commands

```bash
# Verify all pods and their node placement
kubectl get pods -o wide

# Verify nodes are Ready
kubectl get nodes -o wide

# Test pod connectivity (creates a temporary curl pod)
kubectl run curl --image=dareyregistry/curl -it --rm --restart=Never -- sh

# Inside the curl pod
curl -v <pod-ip>:80

# Check service endpoints (empty = label mismatch)
kubectl get endpoints <service-name>

# Check pod labels match service selector
kubectl get pod <pod-name> --show-labels
```

---

## Lessons Learned

1. **Always add the self-referencing security group rule** when setting up
   a K8s cluster on AWS. Without it, all inter-node traffic is blocked
   silently at the hypervisor level.

2. **Diagnose at the right layer.** Start with the simplest test first:
   node-to-node ping. If that fails, the problem is AWS-level, not K8s.

3. **tcpdump is your best friend.** Run it simultaneously on source and
   destination nodes to find exactly where packets are being dropped.

4. **Source/destination check must be disabled** on all worker nodes
   because pod IPs are not registered with AWS — packets with pod IPs
   as source/destination will be dropped otherwise.

5. **Pod CIDR routes are needed at two levels:**
   - AWS VPC route table (so AWS knows which instance owns which pod CIDR)
   - OS routing table on each worker (so the kernel knows where to send packets)

6. **The bridge CNI used in this from-ground-up setup does not install
   cross-node routes automatically.** Production CNI plugins like Calico,
   Flannel, or Weave handle this automatically — which is one major reason
   to use them in real clusters.
