Here’s a clean `cloudflare-pod-README.md` that you can include alongside your manifests. Feel free to tweak paths, domain names, or descriptions to match your structure.

```md
# Cloudflare Tunnel via Kubernetes Pod (cloudflared)  
This document describes what we set up to run a Cloudflare Tunnel inside your Talos Kubernetes cluster, and how the files and directories relate to each other.

---

## 📂 Project Layout

Assume you’re working in a directory, e.g.:

```

~/Projects/cloudflared-k8s-hello-world/
├── README.md
├── hello-world.yaml
├── index.html
└── (this file) cloudflare-pod-README.md

````

Separately, your Talos cluster manifests live e.g. in `~/talos-homelab/`

---

## 🔧 What We Installed & Why

1. **Secret** (`cloudflared-credentials`)  
   Contains the Cloudflare tunnel JSON credential file (named tunnel UUID).  
   This ensures only the cluster (and pods mounting it) can access the private key.

2. **ConfigMap** (`cloudflared-config`)  
   Stores a minimal `config.yml` for cloudflared:
   ```yaml
   tunnel: <YOUR-TUNNEL-UUID>
   credentials-file: /etc/cloudflared/credentials.json

   ingress:
     - hostname: hello.sogos.io
       service: http://hello-world.default.svc.cluster.local:80
     - service: http_status:404
````

The `ingress` rule points to your `hello-world` service via its internal cluster DNS. This decouples from node IPs/ports.

3. **Deployment** (`cloudflared`)
   Runs `cloudflared tunnel run <tunnel-name>` in a pod.

   * Mounts both the Secret and ConfigMap using a *projected volume* into `/etc/cloudflared`.
   * Sets a minimal `securityContext` (drop privileges, non-root, seccomp) to satisfy PodSecurity policies.
   * Exposes container port `20242` for metrics (if desired).

---

## 🚀 How It Works

* On startup, the pod reads `/etc/cloudflared/credentials.json` (the secret) and `/etc/cloudflared/config.yml` (the config).
* It registers and maintains a tunnel with Cloudflare’s edge (QUIC connections).
* Incoming HTTP(s) requests to `hello.sogos.io` from the Cloudflare network are forwarded internally to your `hello-world` service in Kubernetes (via the ingress rule).
* No need for node IPs, port forwarding, or having your Mac always running.

---

## ✅ Steps Summary

1. Copy tunnel JSON to Kubernetes:

   ```bash
   kubectl create secret generic cloudflared-credentials \
     --from-file=credentials.json=~/.cloudflared/<TUNNEL-UUID>.json
   ```

2. Create ConfigMap with `config.yml`:

   ```bash
   kubectl create configmap cloudflared-config \
     --from-literal=config.yml='… your ingress, tunnel, etc …'
   ```

3. Apply the Deployment manifest (with projected volume).
   Ensure the pod transitions to **Running** (check `kubectl get pods -l app=cloudflared`).

4. Verify logs (`kubectl logs`) to see `INF Registered tunnel connection …`.

5. Test externally:

   ```bash
   curl -v https://hello.sogos.io
   ```

---

## 🧾 Troubleshooting & Common Issues

| Symptom                                                    | Likely Cause                              | Fix / Check                                                     |
| ---------------------------------------------------------- | ----------------------------------------- | --------------------------------------------------------------- |
| Pod fails with mount error “not a directory”               | Attempting to mount file over directory   | Use projected volume combining Secret & ConfigMap (as deployed) |
| “Cannot determine default origin certificate path” warning | Missing legacy `cert.pem`; safe to ignore | Your named tunnel JSON is sufficient                            |
| 502 / timeout from external                                | Service isn’t accessible internally       | Check `kubectl get svc hello-world` and pod readiness           |
| CrashLoopBackOff                                           | Bad path, missing file, wrong tunnel name | Inspect `kubectl describe pod` / `kubectl logs --previous`      |

---

## 🧩 Notes & Next Steps

* You can expand this pattern to host multiple apps: just add more ingress entries in `config.yml` (or a custom `configmap`) pointing to other internal services.
* Consider adding RBAC rules or a dedicated ServiceAccount for `cloudflared` if you want stricter access control.
* You may also expose metrics (20242) to Prometheus via ServiceMonitor if you’re running Prometheus in your cluster.
* Monitor Cloudflare Tunnel status and logs over time to catch reconnection or auth issues.

---
