# Talos Homelab - Mac Mini Cluster

A 3-node high-availability Kubernetes cluster running Talos Linux on Mac Mini hardware.

## Cluster Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Talos Homelab Cluster                        │
│                   3-Node HA Control Plane                       │
└─────────────────────────────────────────────────────────────────┘

    ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
    │ mac-mini-1   │      │ mac-mini-2   │      │ mac-mini-3   │
    │ 192.168.1.223│      │ 192.168.1.162│      │ 192.168.1.215│
    │              │      │              │      │              │
    │ Control Plane│      │ Control Plane│      │ Control Plane│
    │ etcd Member  │◄────►│ etcd Member  │◄────►│ etcd Member  │
    │ 63GB RAM     │      │ 63GB RAM     │      │ 63GB RAM     │
    └──────┬───────┘      └──────┬───────┘      └──────┬───────┘
           │                     │                     │
           └─────────────────────┴─────────────────────┘
                              │
                         LAN (enp68s0)
                    Thunderbolt Networking
```

## Cluster Specifications

| Component | Details |
|-----------|---------|
| **Talos Version** | v1.11.2 |
| **Kubernetes Version** | v1.34.0 |
| **CNI** | Flannel |
| **Node Count** | 3 (all control plane) |
| **etcd** | 3-member quorum (HA) |
| **Custom Extensions** | Thunderbolt (v1.11.2) |
| **Network** | Static IPs, Thunderbolt networking |

## Node Information

| Hostname | IP Address | MAC Address | Role | Resources |
|----------|------------|-------------|------|-----------|
| mac-mini-1 | 192.168.1.223 | 00:30:93:12:59:a1 | Control Plane | 63GB RAM, 12-core CPU |
| mac-mini-2 | 192.168.1.162 | 00:30:93:12:54:ce | Control Plane | 63GB RAM, 12-core CPU |
| mac-mini-3 | 192.168.1.215 | 00:30:93:12:59:99 | Control Plane | 63GB RAM, 12-core CPU |

### Network Configuration

- **Primary Interface**: `enp68s0` (Thunderbolt-based 10G)
- **Ignored Interface**: `enp4s0` (onboard Ethernet)
- **Gateway**: 192.168.1.1
- **Subnet**: 192.168.1.0/24

### Hardware

- **External Network Adapter**: Sonnet Technologies Solo 10G SFP+ Thunderbolt 3 Edition
- **Storage**: NVMe (`/dev/nvme0n1`)

## Directory Structure

```
~/talos-k8s-cluster/
├── README.md                      # This file
├── controlplane.yaml              # Control plane machine config
├── worker.yaml                    # Worker machine config (unused)
├── talosconfig                    # Talos client configuration
└── patches/
    ├── mac-mini-patch.yaml        # Base patch (all nodes)
    ├── mac-mini-1-patch.yaml      # Node 1 specific config
    ├── mac-mini-2-patch.yaml      # Node 2 specific config
    ├── mac-mini-3-patch.yaml      # Node 3 specific config
    └── hostname-mac-mini-*.yaml   # Hostname patches
```

## Configuration Files

### Base Patch (mac-mini-patch.yaml)

Applied to all nodes - includes Thunderbolt extension and disk configuration:

```yaml
machine:
  install:
    disk: /dev/nvme0n1
    extensions:
      - image: ghcr.io/siderolabs/thunderbolt:latest
  kernel:
    modules:
      - name: thunderbolt
      - name: thunderbolt_net
  kubelet:
    extraArgs:
      rotate-server-certificates: true
```

### Node-Specific Patches

Each node has a patch with its static IP and hostname configuration:

```yaml
machine:
  install:
    extraKernelArgs:
      - talos.network.interface.ignore=enp4s0
  network:
    hostname: mac-mini-1  # or mac-mini-2, mac-mini-3
    interfaces:
      - interface: enp68s0
        addresses:
          - 192.168.1.223/24  # Unique per node
        routes:
          - network: 0.0.0.0/0
            gateway: 192.168.1.1
```

## Custom Image Factory Schematic

The cluster uses a custom Talos image with the Thunderbolt extension baked in:

- **Schematic ID**: `edf6a4b727a7aa090af375d1e82a198c11c202b272ca9c0d7334cfda7b3220fd`
- **Installer Image**: `factory.talos.dev/metal-installer/edf6a4b727a7aa090af375d1e82a198c11c202b272ca9c0d7334cfda7b3220fd:v1.11.2`
- **ISO**: Available at [factory.talos.dev](https://factory.talos.dev/image/edf6a4b727a7aa090af375d1e82a198c11c202b272ca9c0d7334cfda7b3220fd/v1.11.2/metal-amd64.iso)

### Schematic Configuration

```yaml
customization:
  extraKernelArgs:
    - talos.network.interface.ignore=enp4s0
  systemExtensions:
    officialExtensions:
      - siderolabs/thunderbolt
```

## Environment Variables

Add these to your `~/.zshrc` (or `~/.bashrc`):

```bash
# Talos Homelab Cluster
export CONTROL_PLANE_1_IP=192.168.1.223
export CONTROL_PLANE_2_IP=192.168.1.162
export CONTROL_PLANE_3_IP=192.168.1.215
export CLUSTER_NAME=macmini-cluster
export DISK_NAME=nvme0n1
export TALOSCONFIG=~/.talos/config
```

Reload your shell after adding:
```bash
source ~/.zshrc
```

## Understanding Talos: talosctl vs kubectl

Talos is a **modern OS for Kubernetes** - it has no shell access, no SSH, and is configured entirely via API.

### Two Management Tools

```
┌─────────────────────────────────────────────────────────────┐
│                        Your Workstation                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  talosctl                          kubectl                   │
│  (Talos API)                       (Kubernetes API)          │
│      │                                  │                    │
│      │                                  │                    │
└──────┼──────────────────────────────────┼───────────────────┘
       │                                  │
       ▼                                  ▼
┌─────────────────────────────────────────────────────────────┐
│                      Talos Linux Node                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Talos System Services (apid, trustd, etc.)        │    │
│  │  • OS Configuration                                 │    │
│  │  • Network Setup                                    │    │
│  │  • Disk Management                                  │    │
│  │  • System Extensions                                │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Kubernetes Components (kubelet, etc.)             │    │
│  │  • Pods                                             │    │
│  │  • Deployments                                      │    │
│  │  • Services                                         │    │
│  │  • ConfigMaps, Secrets, etc.                       │    │
│  └────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### talosctl - Manage the OS

**Purpose**: Control Talos Linux itself (the operating system layer)

**Common Commands**:
```bash
# View system logs
talosctl dmesg --nodes $CONTROL_PLANE_1_IP

# Check service status
talosctl service etcd status --nodes $CONTROL_PLANE_1_IP

# Interactive dashboard
talosctl dashboard --nodes $CONTROL_PLANE_1_IP

# Apply configuration changes
talosctl patch machineconfig --nodes $CONTROL_PLANE_1_IP --patch @patch.yaml

# Upgrade Talos OS
talosctl upgrade --nodes $CONTROL_PLANE_1_IP --image factory.talos.dev/...

# Reboot node
talosctl reboot --nodes $CONTROL_PLANE_1_IP

# Check etcd cluster
talosctl etcd members --nodes $CONTROL_PLANE_1_IP

# View network interfaces
talosctl get links --nodes $CONTROL_PLANE_1_IP
```

**Use talosctl for**: OS config, networking, disk management, system services, node operations

### kubectl - Manage Kubernetes

**Purpose**: Control Kubernetes workloads (applications and services)

**Common Commands**:
```bash
# View nodes
kubectl get nodes

# View pods
kubectl get pods -A

# Deploy application
kubectl apply -f deployment.yaml

# View logs
kubectl logs pod-name -n namespace

# Execute commands in pod
kubectl exec -it pod-name -- /bin/sh

# View services
kubectl get services -A

# Check cluster info
kubectl cluster-info
```

**Use kubectl for**: Deploying apps, managing pods, services, ingress, storage, RBAC

### Key Differences

| Aspect | talosctl | kubectl |
|--------|----------|---------|
| **Layer** | Operating System | Kubernetes |
| **Manages** | Talos nodes, networking, system config | Pods, deployments, services |
| **Authentication** | Talos CA certificates | Kubernetes CA certificates |
| **Config File** | `~/.talos/config` or `talosconfig` | `~/.kube/config` |
| **Port** | 50000 (Talos API) | 6443 (Kubernetes API) |

## Post-Installation Setup

### Automatic CSR Approver (Required)

Since kubelet certificate rotation is enabled, you must install an automatic CSR approver to prevent certificate approval warnings:

```bash
kubectl apply -f https://raw.githubusercontent.com/alex1989hu/kubelet-serving-cert-approver/main/deploy/standalone-install.yaml
```

This controller automatically approves kubelet serving certificate signing requests, eliminating the need for manual intervention.

**Verify it's running:**

```bash
kubectl get pods -n kubelet-serving-cert-approver
```

You should see the `kubelet-serving-cert-approver` pod in `Running` state.

## Common Operations

### Check Cluster Health

```bash
# Talos health check
talosctl health --nodes $CONTROL_PLANE_1_IP

# Kubernetes health check
kubectl get nodes
kubectl get pods -A
```

### View Logs

```bash
# System logs (Talos)
talosctl dmesg --follow --nodes $CONTROL_PLANE_1_IP

# Service logs (Talos)
talosctl logs kubelet --follow --nodes $CONTROL_PLANE_1_IP

# Pod logs (Kubernetes)
kubectl logs -f pod-name -n namespace
```

### Reboot Cluster

```bash
# Graceful shutdown all nodes
talosctl shutdown --nodes $CONTROL_PLANE_1_IP,$CONTROL_PLANE_2_IP,$CONTROL_PLANE_3_IP

# Reboot single node
talosctl reboot --nodes $CONTROL_PLANE_1_IP
```

### Monitor Network

```bash
# Check network interfaces
talosctl get links --nodes $CONTROL_PLANE_1_IP

# Verify Thunderbolt module loaded
talosctl read /proc/modules --nodes $CONTROL_PLANE_1_IP | grep thunderbolt

# Check routes
talosctl get routes --nodes $CONTROL_PLANE_1_IP
```

### Update Node Configuration

```bash
# Apply a patch to change config
talosctl patch machineconfig \
  --nodes $CONTROL_PLANE_1_IP \
  --patch @patches/your-patch.yaml
```

### Upgrade Cluster

```bash
# Upgrade Talos OS
talosctl upgrade \
  --nodes $CONTROL_PLANE_1_IP,$CONTROL_PLANE_2_IP,$CONTROL_PLANE_3_IP \
  --image factory.talos.dev/metal-installer/YOUR_SCHEMATIC_ID:v1.11.2 \
  --preserve

# Upgrade Kubernetes
talosctl upgrade-k8s --to 1.34.0
```

## Bootstrapping From Scratch

If you need to rebuild the cluster:

### 1. Download Custom ISO

```bash
curl -LO https://factory.talos.dev/image/edf6a4b727a7aa090af375d1e82a198c11c202b272ca9c0d7334cfda7b3220fd/v1.11.2/metal-amd64.iso
```

### 2. Boot All Nodes from ISO

Create bootable USB and boot all three Mac Minis.

### 3. Set Environment Variables

```bash
export CONTROL_PLANE_1_IP=192.168.1.223
export CONTROL_PLANE_2_IP=192.168.1.162
export CONTROL_PLANE_3_IP=192.168.1.215
export CLUSTER_NAME=macmini-cluster
export DISK_NAME=nvme0n1
```

### 4. Generate Configuration

```bash
talosctl gen config $CLUSTER_NAME https://$CONTROL_PLANE_1_IP:6443 \
  --config-patch @patches/mac-mini-patch.yaml
```

### 5. Apply Configurations

```bash
# Mac Mini 1
talosctl apply-config --insecure \
  --nodes $CONTROL_PLANE_1_IP \
  --file controlplane.yaml \
  --config-patch @patches/mac-mini-1-patch.yaml

# Mac Mini 2
talosctl apply-config --insecure \
  --nodes $CONTROL_PLANE_2_IP \
  --file controlplane.yaml \
  --config-patch @patches/mac-mini-2-patch.yaml

# Mac Mini 3
talosctl apply-config --insecure \
  --nodes $CONTROL_PLANE_3_IP \
  --file controlplane.yaml \
  --config-patch @patches/mac-mini-3-patch.yaml
```

### 6. Configure talosctl

```bash
# Set endpoints
talosctl config endpoint $CONTROL_PLANE_1_IP $CONTROL_PLANE_2_IP $CONTROL_PLANE_3_IP

# Set context
talosctl config context macmini-cluster
```

### 7. Bootstrap etcd

**IMPORTANT**: Run this ONLY ONCE on ONE node!

```bash
talosctl bootstrap --nodes $CONTROL_PLANE_1_IP
```

### 8. Get kubeconfig

```bash
talosctl kubeconfig --nodes $CONTROL_PLANE_1_IP
```

### 9. Verify Cluster

```bash
kubectl get nodes
talosctl health --nodes $CONTROL_PLANE_1_IP
```

### 10. Install Automatic CSR Approver

This prevents certificate rotation warnings:

```bash
kubectl apply -f https://raw.githubusercontent.com/alex1989hu/kubelet-serving-cert-approver/main/deploy/standalone-install.yaml
```

## Troubleshooting

### Nodes Not Coming Online After Reboot

**Check Thunderbolt module**:
```bash
talosctl read /proc/modules --nodes $CONTROL_PLANE_1_IP | grep thunderbolt
```

Should show:
```
thunderbolt_net 49152 0 - Live
thunderbolt 491520 1 thunderbolt_net, Live
```

**Verify network interface**:
```bash
talosctl get links --nodes $CONTROL_PLANE_1_IP
```

`enp68s0` should show `OPER STATE: up`

### etcd Not Forming Quorum

```bash
# Check etcd members
talosctl etcd members --nodes $CONTROL_PLANE_1_IP

# Check etcd service status
talosctl service etcd status --nodes $CONTROL_PLANE_1_IP
```

### Certificate Signing Request (CSR) Issues

If you see a diagnostic about "kubelet server certificate rotation is enabled, but CSR is not approved":

```bash
# Check pending CSRs
kubectl get csr

# Approve all pending CSRs
kubectl get csr -o json | jq -r '.items[] | select(.status == {}) | .metadata.name' | xargs kubectl certificate approve

# Or manually approve each one
kubectl certificate approve <csr-name>
```

**Install automatic CSR approver** (recommended to prevent this issue):

```bash
kubectl apply -f https://raw.githubusercontent.com/alex1989hu/kubelet-serving-cert-approver/main/deploy/standalone-install.yaml
```

This controller automatically approves kubelet serving certificate requests, preventing manual intervention.

### Duplicate Nodes After Hostname Change

```bash
# List all nodes
kubectl get nodes

# Delete old node registrations
kubectl delete node old-node-name
```

## Useful Tools

### Dashboard

```bash
# Talos interactive dashboard
talosctl dashboard --nodes $CONTROL_PLANE_1_IP,$CONTROL_PLANE_2_IP,$CONTROL_PLANE_3_IP

# Install k9s for Kubernetes
brew install k9s
k9s
```

### Log Streaming

```bash
# Install stern for better log streaming
brew install stern

# Stream logs from all pods in namespace
stern -n kube-system .
```

## Network Diagram

```
                          Internet
                              │
                              │
                    ┌─────────▼─────────┐
                    │   Router/Gateway   │
                    │   192.168.1.1      │
                    └─────────┬─────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
    ┌─────────▼─────────┐ ┌──▼────────────┐ ┌▼──────────────┐
    │   mac-mini-1      │ │  mac-mini-2   │ │  mac-mini-3   │
    │   192.168.1.223   │ │  192.168.1.162│ │  192.168.1.215│
    ├───────────────────┤ ├───────────────┤ ├───────────────┤
    │ Thunderbolt Port  │ │Thunderbolt Port│ │Thunderbolt Port│
    │        │          │ │       │        │ │       │       │
    │   ┌────▼────┐    │ │  ┌────▼────┐  │ │  ┌────▼────┐  │
    │   │Sonnet   │    │ │  │Sonnet   │  │ │  │Sonnet   │  │
    │   │10G SFP+ │    │ │  │10G SFP+ │  │ │  │10G SFP+ │  │
    │   └────┬────┘    │ │  └────┬────┘  │ │  └────┬────┘  │
    │        │         │ │       │        │ │       │       │
    │   enp68s0 (UP)  │ │  enp68s0 (UP) │ │  enp68s0 (UP) │
    │   enp4s0 (IGN)  │ │  enp4s0 (IGN) │ │  enp4s0 (IGN) │
    └─────────────────┘ └───────────────┘ └───────────────┘

    Flannel Overlay Network: 10.244.0.0/16
    Pod Network communication across all nodes
```

## Notes

- **No SSH**: Talos has no SSH access. All management is via `talosctl` API.
- **Immutable**: OS is immutable and configured declaratively.
- **Secure Boot**: Custom image supports SecureBoot if needed.
- **etcd Quorum**: 3-node cluster can survive 1 node failure.
- **Thunderbolt Networking**: Critical for Mac Mini networking; baked into custom image.
- **Reboot Safe**: Cluster tested and verified to survive full power cycles.

## Resources

- [Talos Documentation](https://www.talos.dev/)
- [Image Factory](https://factory.talos.dev/)
- [Talos GitHub](https://github.com/siderolabs/talos)
- [Kubernetes Documentation](https://kubernetes.io/docs/)

## Git Workflow

This repository uses a branch-based workflow with automated validation.

### Branch Structure

- **`main`** - Production-ready configurations, protected branch
- **`dev`** - Development and testing branch

### Making Changes

```bash
# 1. Start from latest main
git checkout main
git pull origin main

# 2. Create/switch to dev branch
git checkout -b dev

# 3. Make your changes
vim patches/new-feature.yaml

# 4. Commit changes
git add patches/new-feature.yaml
git commit -m "Add new feature configuration"

# 5. Push to dev branch
git push origin dev

# 6. Create Pull Request on GitHub (dev → main)
# GitHub will automatically run validation checks

# 7. After PR is merged, sync your local main
git checkout main
git pull origin main

# 8. Clean up and start fresh for next change
git branch -d dev
git checkout -b dev
git push -u origin dev
```

### Automated Validation

GitHub Actions automatically validates all YAML files on:
- Every push to any branch
- Every pull request

**Validation checks:**
- YAML syntax correctness
- File structure validation

The workflow runs `yamllint` on all files in the `patches/` directory.

### Branch Protection (Optional)

To enforce validation before merging to `main`:

1. Go to: Settings → Branches → Add rule
2. Branch name pattern: `main`
3. Enable:
   - ☑️ Require a pull request before merging
   - ☑️ Require status checks to pass before merging
   - Select: `validate` workflow

With branch protection enabled, you **cannot merge** to `main` unless all checks pass.

### What's Ignored

The following files are **not** committed to Git (sensitive data):

```
talosconfig          # Contains authentication certificates
controlplane.yaml    # Contains cluster secrets
worker.yaml          # Contains cluster secrets
*.key                # Private keys
```

These files remain local only and are listed in `.gitignore`.

### Quick Commands

```bash
# Check what's changed
git status

# View commit history
git log --oneline

# See what's ignored
git status --ignored

# View difference before committing
git diff

# Undo uncommitted changes
git checkout -- <file>

# Pull latest from remote
git pull origin main
```

## Cluster Status

✅ **Production Ready**
- All nodes operational
- etcd quorum healthy
- Thunderbolt networking functional
- Survives reboots gracefully
- Static IPs persistent
- Custom extensions loaded

---

*Last Updated: October 13, 2025*
*Cluster Built: October 13, 2025*
