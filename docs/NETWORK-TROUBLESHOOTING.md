# Kubernetes Network Troubleshooting Guide

This guide documents common network issues in Kubernetes clusters and systematic troubleshooting steps, based on a real-world Flannel CNI networking failure in a Talos Linux cluster.

## Table of Contents
- [Quick Reference Commands](#quick-reference-commands)
- [Systematic Troubleshooting Process](#systematic-troubleshooting-process)
- [Case Study: Flannel VXLAN Route Failure](#case-study-flannel-vxlan-route-failure)
- [Common Network Issues](#common-network-issues)
- [Prevention and Monitoring](#prevention-and-monitoring)

---

## Quick Reference Commands

### Cluster Overview
```bash
# Get all namespaces
kubectl get namespaces

# Get all pods across all namespaces with node placement
kubectl get pods -A -o wide

# Get all services
kubectl get svc -A

# Get all deployments
kubectl get deploy -A

# Get all network policies
kubectl get networkpolicy -A

# Get node status
kubectl get nodes -o wide

# Comprehensive cluster view
kubectl get all,cm,secret,networkpolicy -A -o wide
```

### Pod and Service Inspection
```bash
# Describe a specific pod
kubectl describe pod <pod-name> -n <namespace>

# Get pod logs
kubectl logs <pod-name> -n <namespace>
kubectl logs -l app=<label> --tail=100

# Get logs from previous crashed container
kubectl logs <pod-name> --previous

# Check service endpoints
kubectl get endpoints <service-name> -n <namespace>

# Describe a service
kubectl describe svc <service-name> -n <namespace>
```

### DNS Testing
```bash
# Test DNS resolution
kubectl run test-dns --image=busybox --rm -it --restart=Never -- nslookup <service-name>.<namespace>.svc.cluster.local

# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
```

### Network Connectivity Testing
```bash
# Test TCP connectivity to a service
kubectl run test-tcp --image=busybox --rm -it --restart=Never -- wget -O- --timeout=5 http://<service-name>.<namespace>.svc.cluster.local:<port>

# Test connectivity to pod IP directly
kubectl run test-direct --image=busybox --rm -it --restart=Never -- wget -O- --timeout=5 http://<pod-ip>:<port>

# Test UDP connectivity (for DNS)
kubectl run udp-test --image=busybox --rm -it --restart=Never -- sh -c "echo 'test' | nc -u -w 1 <pod-ip> 53"

# Test external connectivity
kubectl run test-external --image=busybox --rm -it --restart=Never -- ping -c 3 8.8.8.8
```

---

## Systematic Troubleshooting Process

### Step 1: Identify the Symptom
Document what's broken:
- Which services are affected?
- What error messages appear? (502, 503, timeouts, DNS failures)
- Is it affecting all pods or specific nodes?
- When did it start working incorrectly?

### Step 2: Check Pod Status
```bash
# Look for pods not in Running state
kubectl get pods -A | grep -v Running | grep -v Completed

# Check specific application pods
kubectl get pods -l app=<your-app> -o wide

# Describe problematic pods
kubectl describe pod <pod-name>
```

### Step 3: Test DNS Resolution
```bash
# Test if CoreDNS is running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Test DNS from a pod
kubectl run test-dns --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default

# Check if pods can reach CoreDNS IP (usually 10.96.0.10)
kubectl run test-dns-ip --image=busybox --rm -it --restart=Never -- nslookup google.com 10.96.0.10
```

**Common DNS Issues:**
- `connection timed out; no servers could be reached` → CoreDNS not reachable
- `server misbehaving` → CoreDNS overloaded or misconfigured
- `NXDOMAIN` → Service/domain doesn't exist

### Step 4: Test Service Connectivity
```bash
# Check if service has endpoints
kubectl get endpoints <service-name>

# Test connectivity to service ClusterIP
kubectl run test-svc --image=busybox --rm -it --restart=Never -- wget -O- --timeout=5 http://<service-name>.<namespace>.svc.cluster.local:<port>

# Test connectivity to pod IP directly (bypass service)
kubectl run test-pod --image=busybox --rm -it --restart=Never -- wget -O- --timeout=5 http://<pod-ip>:<port>
```

### Step 5: Check CNI (Container Network Interface)
For Flannel:
```bash
# Check Flannel DaemonSet status
kubectl get ds -n kube-system kube-flannel

# Check Flannel pods
kubectl get pods -n kube-system -l app=flannel -o wide

# Check Flannel logs
kubectl logs -n kube-system -l app=flannel --tail=100
```

For Talos Linux with Flannel:
```bash
# Check routes on each node
for node in <node-ip-1> <node-ip-2> <node-ip-3>; do
  echo "=== Routes on $node ==="
  talosctl -n $node get routes | grep -E "10.244|flannel"
done

# Check Flannel interface status
talosctl -n <node-ip> get links | grep flannel
```

### Step 6: Check Network Policies
```bash
# List all network policies
kubectl get networkpolicy -A

# Describe specific network policy
kubectl describe networkpolicy <policy-name> -n <namespace>
```

Network policies can block traffic between pods. Look for:
- Ingress rules that might block incoming traffic
- Egress rules that might block outgoing traffic

### Step 7: Check kube-proxy
```bash
# Check kube-proxy pods
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# Check kube-proxy logs
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=50
```

---

## Case Study: Flannel VXLAN Route Failure

### Problem Description

**Symptoms:**
- Website accessible via Cloudflare Tunnel returning 502 Bad Gateway errors
- Intermittent connectivity - worked briefly then failed
- Cloudflare Tunnel connected successfully (4 edge connections)
- Error in logs: `dial udp 10.244.3.14:53920->10.96.0.10:53: i/o timeout`

**Environment:**
- 3-node Talos Linux cluster (Mac Minis in HA)
- Kubernetes v1.34.0
- Flannel CNI with VXLAN backend
- Cloudflare Tunnel for external access

### Investigation Steps

#### 1. Initial Symptoms
```bash
# Cloudflare tunnel logs showed DNS timeouts
kubectl logs -l app=cloudflared --tail=50
# Error: "dial tcp: lookup argocd-server.argocd.svc.cluster.local on 10.96.0.10:53: read udp: i/o timeout"
```

#### 2. DNS Testing
```bash
# Test DNS resolution from test pod
kubectl run test-dns --image=busybox --rm -it --restart=Never -- nslookup mirai-frontend.default.svc.cluster.local
# Result: connection timed out; no servers could be reached
```

**Finding:** DNS completely broken - pods couldn't reach CoreDNS at all.

#### 3. Service Connectivity Testing
```bash
# Test service connectivity
kubectl run test-connectivity --image=busybox --rm -it --restart=Never -- wget -O- --timeout=5 http://mirai-frontend.default.svc.cluster.local:80
# Result: wget: download timed out

# Check service endpoints
kubectl get endpoints mirai-frontend
# Result: Endpoints exist (10.244.3.13:3000, 10.244.5.15:3000)
```

**Finding:** Services had valid endpoints, but pods couldn't communicate.

#### 4. Check CoreDNS Status
```bash
# CoreDNS pods running
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
# NAME                       IP            NODE
# coredns-7bb49dc74c-57phq   10.244.5.2    mac-mini-3
# coredns-7bb49dc74c-k9nq2   10.244.4.13   mac-mini-2

# CoreDNS logs showed no errors
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
# Result: Normal startup, no errors
```

**Finding:** CoreDNS was healthy but not receiving requests.

#### 5. TCP vs UDP Testing
```bash
# Test TCP connectivity (worked!)
kubectl run test-tcp --image=busybox --rm -it --restart=Never -- wget -O- --timeout=5 http://10.244.5.2:8080
# Result: HTTP/1.1 404 Not Found (connection successful)

# Test UDP connectivity (failed!)
kubectl run udp-test --image=busybox --rm -it --restart=Never -- sh -c "echo 'test' | nc -u -w 1 10.244.5.2 53"
# Result: timeout
```

**Finding:** TCP worked, UDP failed - suggesting routing or CNI issue, not firewall.

#### 6. Check Flannel Routes (CRITICAL DISCOVERY)
```bash
# Check routes on all nodes
for node in 192.168.1.223 192.168.1.162 192.168.1.215; do
  echo "=== Routes on $node ==="
  talosctl -n $node get routes | grep -E "10.244|flannel"
done
```

**Critical Finding:**
```
=== Routes on 192.168.1.223 (mac-mini-1) ===
10.244.3.0/24  -> cni0 (local)
10.244.4.0/24  -> flannel.1 ✓
10.244.5.0/24  -> flannel.1 ✓

=== Routes on 192.168.1.162 (mac-mini-2) === ← PROBLEM!
10.244.4.0/24  -> cni0 (local only)
NO ROUTES TO 10.244.3.0/24 or 10.244.5.0/24! ✗

=== Routes on 192.168.1.215 (mac-mini-3) ===
10.244.5.0/24  -> cni0 (local)
10.244.3.0/24  -> flannel.1 ✓
10.244.4.0/24  -> flannel.1 ✓
```

**Root Cause Identified:** mac-mini-2 was missing Flannel VXLAN routes to other nodes!

#### 7. Check Flannel Pod Status
```bash
# Flannel logs on mac-mini-2
kubectl logs -n kube-system kube-flannel-dswk4 --tail=200
```

**Finding:** Flannel pod logs showed:
```
I1013 21:58:50.312019 Received Subnet Event with VxLan: BackendType: vxlan, PublicIP: 192.168.1.223
I1013 21:58:50.314066 Received Subnet Event with VxLan: BackendType: vxlan, PublicIP: 192.168.1.215
I1013 21:58:50.343338 iptables.go:357] bootstrap done
I1013 21:58:50.350416 main.go:488] Waiting for all goroutines to exit  ← CRASHED!
```

Flannel received the subnet events but crashed immediately after initialization, before creating the routes.

### Solution

#### Fix Applied
```bash
# Restart the failed Flannel pod on mac-mini-2
kubectl delete pod -n kube-system kube-flannel-dswk4

# Wait for new pod to start
kubectl get pods -n kube-system -l app=flannel -w
```

#### Verification
```bash
# Check routes again
for node in 192.168.1.223 192.168.1.162 192.168.1.215; do
  echo "=== Routes on $node ==="
  talosctl -n $node get routes | grep -E "10.244|flannel"
done
```

**Result After Fix:**
```
=== Routes on 192.168.1.162 (mac-mini-2) === ← FIXED!
10.244.4.0/24  -> cni0 (local)
10.244.3.0/24  -> flannel.1 ✓
10.244.5.0/24  -> flannel.1 ✓
```

#### Test Services
```bash
# Test connectivity
curl -I https://mirai.sogos.io
curl -I https://argocd.sogos.io

# Both returned HTTP/2 200 ✓
```

### Root Cause Analysis

**What Happened:**
1. The cluster was rebooted 7 days prior
2. The Flannel pod on mac-mini-2 crashed during initialization
3. The pod remained in a "running but non-functional" state
4. No VXLAN routes were created to peer nodes
5. Pods on mac-mini-2 couldn't reach CoreDNS or any services on other nodes
6. This caused DNS timeouts and service connectivity failures

**Why It Was Intermittent:**
- When the cloudflared pod ran on mac-mini-1 or mac-mini-3, it worked
- When it ran on mac-mini-2, it failed
- Kubernetes scheduler randomly placed pods across nodes

**Why TCP Worked but UDP Failed:**
Actually, both failed for cross-node communication. The TCP test that "worked" was only returning a 404 error because it couldn't complete the full request - the connection was timing out on the application layer.

### Lessons Learned

1. **Check CNI Routes First:** When experiencing cluster-wide connectivity issues, always check if CNI routes are properly established on all nodes.

2. **Pod Status Isn't Enough:** A pod showing "Running" doesn't mean it's functional. Check logs and validate actual behavior.

3. **Test Node-by-Node:** Network issues can be node-specific. Test connectivity from pods on each node individually.

4. **DNS Timeouts Often Mean Routing Issues:** DNS runs on UDP/53. If DNS is timing out but CoreDNS is healthy, it's usually a routing/CNI problem, not a DNS problem.

5. **Flannel Can Fail Silently:** Flannel can crash during initialization but leave the pod in "Running" state without proper routes configured.

---

## Common Network Issues

### Issue 1: Pod Cannot Resolve DNS

**Symptoms:**
- `nslookup` fails with timeout
- Applications fail with "unknown host" errors

**Troubleshooting:**
```bash
# Check CoreDNS is running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Test DNS directly
kubectl run test-dns --image=busybox --rm -it --restart=Never -- nslookup kubernetes.default
```

**Common Causes:**
- CoreDNS pods not running
- CoreDNS service ClusterIP incorrect
- Network policies blocking port 53
- CNI routing issues (see Case Study)

### Issue 2: Service Returns 502/503 Errors

**Symptoms:**
- Ingress/LoadBalancer returns 502 Bad Gateway
- Service accessible sometimes but not always

**Troubleshooting:**
```bash
# Check backend pods are running
kubectl get pods -l app=<your-app>

# Check service endpoints exist
kubectl get endpoints <service-name>

# Test direct pod connectivity
kubectl run test --image=busybox --rm -it --restart=Never -- wget http://<pod-ip>:<port>
```

**Common Causes:**
- No healthy backend pods
- Pods not matching service selector
- Readiness probes failing
- Network policies blocking traffic
- CNI routing issues

### Issue 3: Pods Can't Communicate Across Nodes

**Symptoms:**
- Pods on same node can communicate
- Pods on different nodes cannot communicate

**Troubleshooting:**
```bash
# Check CNI pods (Flannel example)
kubectl get pods -n kube-system -l app=flannel

# Check routes on each node
talosctl -n <node-ip> get routes | grep flannel

# Test cross-node connectivity
kubectl exec -it <pod-on-node-1> -- ping <pod-ip-on-node-2>
```

**Common Causes:**
- CNI not running on all nodes
- VXLAN/overlay routes not established
- Firewall blocking overlay traffic (UDP 4789 for Flannel)
- MTU mismatches causing packet fragmentation

### Issue 4: External Services Unreachable

**Symptoms:**
- Pods can't reach internet
- `wget google.com` times out

**Troubleshooting:**
```bash
# Test external connectivity
kubectl run test-ext --image=busybox --rm -it --restart=Never -- ping -c 3 8.8.8.8

# Check if DNS works for external domains
kubectl run test-dns --image=busybox --rm -it --restart=Never -- nslookup google.com
```

**Common Causes:**
- No default gateway configured
- Network policies blocking egress
- NAT/masquerading not configured
- Firewall blocking outbound traffic

---

## Prevention and Monitoring

### Regular Health Checks
```bash
# Create a monitoring script
cat > check-cluster-network.sh <<'EOF'
#!/bin/bash
echo "=== Checking CoreDNS ==="
kubectl get pods -n kube-system -l k8s-app=kube-dns

echo "=== Checking CNI ==="
kubectl get pods -n kube-system -l app=flannel

echo "=== Testing DNS ==="
kubectl run test-dns-$(date +%s) --image=busybox --rm -i --restart=Never -- nslookup kubernetes.default 2>&1 | grep -E "Server|Address|Name"

echo "=== Checking Service Endpoints ==="
kubectl get endpoints -A | grep -v "<none>"

echo "=== Node Status ==="
kubectl get nodes
EOF

chmod +x check-cluster-network.sh
```

### Monitor CNI Health
```bash
# Set up alerts for CNI pod failures
kubectl get events -A --field-selector involvedObject.kind=Pod,reason=Failed -w

# Monitor Flannel specifically
kubectl logs -n kube-system -l app=flannel --tail=10 --timestamps
```

### Automated Testing
Create a test pod that continuously monitors connectivity:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: network-monitor
  namespace: default
spec:
  containers:
  - name: monitor
    image: busybox
    command: ["/bin/sh", "-c"]
    args:
    - |
      while true; do
        echo "Testing DNS..."
        nslookup kubernetes.default
        echo "Testing external..."
        wget -O- -T 5 http://google.com > /dev/null 2>&1 && echo "External: OK" || echo "External: FAIL"
        sleep 60
      done
```

### Post-Restart Checklist
After any cluster restart or node reboot:
1. Verify all CNI pods are running
2. Check routes are established on all nodes
3. Test DNS resolution
4. Test cross-node pod communication
5. Verify service endpoints are populated
6. Test external connectivity

---

## Useful Diagnostic Commands

### Talos-Specific Commands
```bash
# Check network interfaces
talosctl -n <node-ip> get links

# Check routes
talosctl -n <node-ip> get routes

# Check system logs
talosctl -n <node-ip> dmesg | grep -i network

# Check running services
talosctl -n <node-ip> service list
```

### Advanced Debugging
```bash
# Capture network traffic
kubectl run tcpdump --image=corfr/tcpdump --rm -it --restart=Never -- tcpdump -i eth0 -n

# Check iptables rules (if accessible)
kubectl run iptables --image=nicolaka/netshoot --rm -it --restart=Never -- iptables -L -n -v

# Full network diagnostic pod
kubectl run netshoot --image=nicolaka/netshoot --rm -it --restart=Never -- bash
```

---

## Summary

Network troubleshooting in Kubernetes requires a systematic approach:

1. **Start at the application layer** - check if pods are running and healthy
2. **Test DNS** - most issues manifest as DNS failures
3. **Verify service connectivity** - check endpoints and test direct pod IPs
4. **Inspect CNI** - ensure routing is correct on all nodes
5. **Check network policies** - verify traffic isn't being blocked
6. **Validate kube-proxy** - ensure service routing rules are correct

Remember: A pod showing "Running" doesn't guarantee it's functional. Always verify behavior with actual connectivity tests.

The case study demonstrates that CNI issues can be subtle and node-specific, requiring careful investigation of routing tables and CNI logs to identify and resolve.

---

**Version:** 1.0  
**Last Updated:** October 21, 2025  
**Cluster:** Talos Linux 1.11.2, Kubernetes 1.34.0, Flannel CNI
