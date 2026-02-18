# vCluster Autonodes Proxmox

This repository contains a Terraform configuration for automatically provisioning virtual machines in Proxmox for use with vCluster Platform's auto-nodes feature. When a vCluster requests a new worker node, vCluster Platform runs this Terraform (via OpenTofu) to create a Proxmox VM, which then joins the vCluster automatically.

## How It Works

```
vCluster (needs node)
  → vCluster Platform detects the pending NodeClaim
    → OpenTofu runs node/main.tf with injected var.vcluster
      → Proxmox VM is created with cloud-init join script
        → VM registers as a worker node in the vCluster
```

vCluster Platform handles all orchestration. You provide the Terraform template and a Kubernetes Secret with Proxmox credentials; the Platform handles everything else.

## Prerequisites

- **vCluster Platform** installed and running in a Kubernetes cluster
- **vCluster CLI** (`vcluster`) connected to the Platform
- **Proxmox Virtual Environment** accessible from the Platform cluster
- **Proxmox API credentials** — either a user/password or an API token (recommended)
- Ubuntu noble cloud image uploaded to Proxmox (`local:iso/noble-server-cloudimg-amd64.img`)
- **Snippets enabled** on the Proxmox `local` datastore (required for cloud-init)

---

## Step 1 — Proxmox Setup

### Upload the Ubuntu Cloud Image

Download the Ubuntu 24.04 (noble) server cloud image and upload it to your Proxmox node:

```bash
# On your Proxmox node
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
# Upload via Proxmox UI: Datacenter → local → ISO Images → Upload
# or with pvesh:
pvesh create /nodes/<node>/storage/local/upload \
  --content iso \
  --filename noble-server-cloudimg-amd64.img \
  --tmpfilename /path/to/noble-server-cloudimg-amd64.img
```

### Enable Snippets on the Local Datastore

In the Proxmox UI: **Datacenter → Storage → local → Edit** and add `Snippets` to the Content types. This is required for cloud-init user-data files.

### Create a Proxmox API Token (Recommended)

```bash
# In Proxmox UI: Datacenter → Permissions → API Tokens → Add
# User: terraform@pam, Token ID: mytoken
# Uncheck "Privilege Separation" if you want it to inherit user permissions

# Grant required permissions:
pveum aclmod / -user terraform@pam -role PVEVMAdmin
pveum aclmod /storage/local -user terraform@pam -role PVEDatastoreAdmin
pveum aclmod /storage/local-lvm -user terraform@pam -role PVEDatastoreAdmin
```

---

## Step 2 — Kubernetes Secret for Credentials

vCluster Platform injects Proxmox credentials into the Terraform run as environment variables via a Kubernetes Secret. The Secret must be in the namespace where vCluster Platform runs and must have a label matching the NodeProvider name.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: proxmox-credentials
  namespace: vcluster-platform   # adjust to your Platform namespace
  labels:
    terraform.vcluster.com/provider: proxmox   # must match the NodeProvider name
type: Opaque
stringData:
  PROXMOX_VE_ENDPOINT: "https://192.168.86.5:8006/"   # your Proxmox URL
  PROXMOX_VE_INSECURE: "true"                          # set to "false" with valid certs

  # Option A: Username + password
  PROXMOX_VE_USERNAME: "terraform@pam"
  PROXMOX_VE_PASSWORD: "your-password"

  # Option B: API token (recommended — comment out Option A)
  # PROXMOX_VE_API_TOKEN: "terraform@pam!mytoken=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

Apply with:

```bash
kubectl apply -f proxmox-credentials.yaml
```

The `bpg/proxmox` provider reads these environment variables automatically. No credentials are hardcoded in `main.tf`.

---

## Step 3 — Create the NodeProvider

The NodeProvider tells vCluster Platform where to find the Terraform template and defines the available node sizes (nodeTypes).

Create a file `nodeprovider.yaml`:

```yaml
apiVersion: management.loft.sh/v1
kind: NodeProvider
metadata:
  name: proxmox   # must match the label value on your credentials Secret
spec:
  terraform:
    nodeTemplate:
      git:
        repository: https://github.com/your-org/vcluster-autonodes-proxmox.git
        branch: main
        subPath: node   # points to the directory containing main.tf
    nodeTypes:
      - name: small
        resources:
          cpu: "2"
          memory: "4096M"
        maxCapacity: 10
      - name: medium
        resources:
          cpu: "4"
          memory: "8192M"
        maxCapacity: 5
      - name: large
        resources:
          cpu: "8"
          memory: "16384M"
        maxCapacity: 3
```

Apply with:

```bash
kubectl apply -f nodeprovider.yaml
```

> **Note on `subPath`**: The `subPath: node` field tells the Platform to use the `node/` subdirectory of the repository as the Terraform root. This is where `main.tf` lives.

---

## Step 4 — Create a vCluster with Auto-Nodes

Add `privateNodes` to your vCluster configuration to enable auto-node provisioning. You can put this in a vCluster Platform template or pass it directly when creating a vCluster.

### vCluster Template or Values YAML

```yaml
controlPlane:
  service:
    spec:
      type: LoadBalancer
privateNodes:
  enabled: true
  autoNodes:
    - provider: proxmox
      nodeType: small       # must match a nodeType name in your NodeProvider
      dynamic:
        - name: workload-pool
          limits:
            nodes: 10
networking:
  podCIDR: 10.64.0.0/16
  serviceCIDR: 10.128.0.0/16
```

When this vCluster needs a new node (e.g., due to pending pods), the Platform creates a NodeClaim, which triggers a Terraform run using the NodeProvider configuration.

---

## How `var.vcluster` Works

The Terraform configuration uses `var.vcluster` throughout — for example:

```hcl
var.vcluster.nodeClaim.metadata.name   # unique name for this node claim
var.vcluster.nodeType.spec.resources.cpu    # CPU cores from the nodeType
var.vcluster.nodeType.spec.resources.memory # memory (e.g., "4096M")
var.vcluster.userData                       # cloud-init join script from Platform
```

**You do not define `var.vcluster` externally.** vCluster Platform injects it automatically into every Terraform run. The variable is declared in `main.tf` as `type = any` so it accepts the full object the Platform provides.

---

## Customizing `main.tf`

### Proxmox Node Name

The `node_name = "ai"` in both resources refers to the Proxmox host node. Change this to match your Proxmox node name:

```hcl
node_name = "pve1"   # your Proxmox node name
```

### Disk Size

The default disk size is 120 GB. Adjust the `size` field in the `disk` block:

```hcl
disk {
  size = 50   # GB
  ...
}
```

### Static IP Addressing

Replace the DHCP `ip_config` block with a static configuration:

```hcl
ip_config {
  ipv4 {
    address = "192.168.86.100/24"
    gateway = "192.168.86.1"
  }
}
```

---

## Security Considerations

### API Token vs Password

Use an API token instead of a username/password. Tokens can be scoped to specific permissions and revoked independently:

```
PROXMOX_VE_API_TOKEN: "terraform@pam!mytoken=<uuid>"
```

### SSH / Console Access

The default `main.tf` enables the `ubuntu` user with password `ubuntu` for console/SSH access during development. **Disable this in production:**

```yaml
# Remove or replace with SSH key authentication:
users:
  - name: ubuntu
    ssh_authorized_keys:
      - "ssh-ed25519 AAAA... your-key"
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
```

### TLS Certificates

`PROXMOX_VE_INSECURE: "true"` skips TLS verification. For production, use a valid certificate on your Proxmox instance and set this to `"false"` (or omit it entirely).

### Secret Namespace

The credentials Secret should be in the same namespace as vCluster Platform. Restrict access to this namespace using Kubernetes RBAC to limit who can read the Secret.

---

## Troubleshooting

| Symptom | Likely Cause |
|---|---|
| `Error: Missing required argument "endpoint"` | `PROXMOX_VE_ENDPOINT` not set in the Secret |
| `Error: Unsupported argument` on `insecure` | Move `insecure` to env var `PROXMOX_VE_INSECURE` in the Secret |
| `Error: variable "vcluster" is not declared` | Old version of main.tf without the `variable "vcluster"` block |
| `Error: Provider "random" not found` | Old version of main.tf missing `random` in `required_providers` |
| Snippets upload fails | Snippets not enabled on the `local` datastore |
| VM created but never joins cluster | Cloud-init userData not applied — check `PROXMOX_VE_ENDPOINT` reachability from VM |
| VM name collision | The `random_string` suffix prevents this; if collisions occur, check if VMs are being cleaned up |
