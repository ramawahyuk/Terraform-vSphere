# 🏗️ terraform-vsphere-rhel9-automation

> **Terraform + Ansible integration for automated RHEL 9 VM provisioning on VMware vSphere**  
> Clone a RHEL 9 template, provision one or many VMs, and automatically trigger Ansible hardening — all from a single `terraform apply`.

[![Terraform](https://img.shields.io/badge/Terraform-%3E%3D1.5.0-7B42BC?logo=terraform)](https://www.terraform.io/)
[![vSphere Provider](https://img.shields.io/badge/vSphere_Provider-~%3E2.6-blue?logo=vmware)](https://registry.terraform.io/providers/hashicorp/vsphere/latest)
[![RHEL](https://img.shields.io/badge/OS-RHEL_9-red?logo=redhat)](https://www.redhat.com/)
[![Ansible](https://img.shields.io/badge/Ansible-auto--triggered-red?logo=ansible)](https://www.ansible.com/)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Phase 4 Objectives](#phase-4-objectives)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [How Terraform Triggers Ansible](#how-terraform-triggers-ansible)
- [Variables Reference](#variables-reference)
- [Outputs](#outputs)
- [Drift Detection](#drift-detection)
- [Troubleshooting](#troubleshooting)

---

## Overview

This project implements **Phase 4** of a VMware home lab automation series. It extends a basic Terraform vSphere setup to support:

- Cloning multiple RHEL 9 VMs from a vSphere template in a single run
- Automatically triggering Ansible on a dedicated control node after each VM is provisioned
- A fully hands-free pipeline: one command delivers a hardened, configured VM

**The complete automated flow:**

```
terraform apply
      │
      ▼
Terraform clones RHEL9-Template in vSphere
      │
      ▼
VM is powered on, IP assigned via open-vm-tools
      │
      ▼
Terraform local-exec provisioner SSH → Ansible control node
      │
      ▼
Ansible runs site.yml against the new VM's IP
      │
      ▼
VM is hardened and configured — fully hands-free
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Windows PC (Terraform runs here)                           │
│  C:\terraform\vsphere-starter\                              │
│                                                             │
│  terraform apply                                            │
│       │                                                     │
│       ▼  vSphere API (HTTPS/443)                           │
├─────────────────────────────────────────────────────────────┤
│  VMware ESXi Host (192.168.198.X)                           │
│  vCenter/vSphere                                            │
│                                                             │
│  ┌──────────────┐    clone    ┌──────────────────────┐     │
│  │ RHEL9-Template│ ─────────► │ rhel9-lab-01         │     │
│  │ (template)   │            │ rhel9-lab-02  etc.   │     │
│  └──────────────┘            └──────────┬───────────┘     │
│                                         │ IP assigned      │
├─────────────────────────────────────────┼───────────────────┤
│  RHEL 9 Ansible Control Node            │                   │
│  (192.168.198.134)                      │                   │
│  ansible_svc user                       │                   │
│                                         ▼                   │
│  ansible-playbook site.yml ──────► new VM hardened         │
└─────────────────────────────────────────────────────────────┘
```

---

## Phase 4 Objectives

| # | Objective | Status |
|---|-----------|--------|
| 1 | Clone multiple VMs with Terraform (`count`) | ✅ |
| 2 | Auto-trigger Ansible after Terraform provisions | ✅ |
| 3 | CI/CD pipeline with GitLab CI | 🔜 Phase 5 |
| 4 | Drift detection (`terraform plan` scheduled) | 🔜 Phase 5 |

---

## Prerequisites

### Tools (Windows PC / Terraform host)

- [Terraform >= 1.5.0](https://www.terraform.io/downloads)
- SSH client accessible from terminal (Git Bash, WSL, or OpenSSH)
- Network access to ESXi host on port 443

### vSphere Environment

- VMware ESXi or vCenter (standalone ESXi supported)
- RHEL 9 VM converted to template (`RHEL9-Template`)
- `open-vm-tools` installed in the template (required for IP reporting)

### RHEL 9 Ansible Control Node

- Dedicated RHEL 9 VM at a static IP (e.g. `192.168.198.134`)
- `ansible_svc` service account with sudo access
- SSH key pair at `/home/ansible_svc/.ssh/ansible_id_rsa`
- Ansible installed, playbook at `/home/ansible_svc/ansible/rhel-lab/site.yml`

### Template Preparation (Step 1)

In vSphere Web UI:

```
Right-click your RHEL 9 VM (192.168.198.134)
→ Clone
→ Clone as Template
→ Name: RHEL9-Template
```

Ensure the template has:
- `open-vm-tools` installed (`dnf install open-vm-tools -y`)
- `ansible_svc` user with SSH public key pre-authorized
- Network adapter configured (VM Network)

---

## Project Structure

```
C:\terraform\vsphere-starter\
│
├── main.tf                  ← Provider, data sources, VM resource, local-exec provisioner
├── variables.tf             ← All input variable declarations with types and defaults
├── terraform.tfvars         ← Your real values (never commit passwords)
├── outputs.tf               ← VM names, IPs, and IDs after apply
│
└── scripts\
    └── provision.sh         ← Shell script that SSH → Ansible control node and runs playbook
```

---

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/<your-username>/terraform-vsphere-rhel9-automation.git
cd terraform-vsphere-rhel9-automation
```

### 2. Copy and Edit tfvars

```bash
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars with your real ESXi IP, credentials, and VM settings
```

### 3. Set the vSphere Password Securely

**PowerShell (Windows):**
```powershell
$env:TF_VAR_vsphere_password = "YourPasswordHere"
```

**Bash (Linux/WSL):**
```bash
export TF_VAR_vsphere_password="YourPasswordHere"
```

> Never put the password in `terraform.tfvars` or any committed file.

### 4. Initialize, Plan, Apply

```bash
# Download the vSphere provider (~2.6)
terraform init

# Preview what will be created
terraform plan

# Provision VMs and trigger Ansible
terraform apply
# type: yes when prompted
```

### 5. Verify

After apply completes, Terraform outputs the VM names and IPs:

```
vm_names        = ["rhel9-lab-01"]
vm_ip_addresses = ["192.168.198.135"]
vm_ids          = ["422f..."]
```

Check vSphere Web UI — the VM should appear under your datacenter, powered on.

---

## Configuration

### `terraform.tfvars` Key Values

| Variable | Example | Description |
|----------|---------|-------------|
| `vsphere_server` | `192.168.198.X` | ESXi or vCenter IP |
| `vsphere_user` | `administrator@vsphere.local` | vSphere admin account |
| `datacenter_name` | `ha-datacenter` | Must match exactly in vSphere |
| `datastore_name` | `datastore1` | Datastore for VM disks |
| `network_name` | `VM Network` | Port group name |
| `esxi_host_name` | `192.168.198.X` | Same as server for standalone ESXi |
| `template_name` | `RHEL9-Template` | Must match exactly in vSphere |
| `vm_count` | `1` | Number of VMs to clone |
| `vm_prefix` | `rhel9-lab` | Produces `rhel9-lab-01`, `rhel9-lab-02`, etc. |
| `vm_cpu` | `2` | vCPUs per VM |
| `vm_ram` | `2048` | RAM in MB per VM |
| `vm_disk_size` | `40` | Disk in GB (thin provisioned) |
| `ansible_user` | `ansible_svc` | SSH user for Ansible control node |
| `ansible_key_path` | `/home/ansible_svc/.ssh/ansible_id_rsa` | SSH key path on control node |
| `ansible_playbook_path` | `/home/ansible_svc/ansible/rhel-lab/site.yml` | Playbook path on control node |

### Scaling to Multiple VMs

To create 3 VMs at once, change in `terraform.tfvars`:

```hcl
vm_count = 3
```

Terraform will create `rhel9-lab-01`, `rhel9-lab-02`, `rhel9-lab-03` and trigger Ansible against each VM's IP independently.

---

## How Terraform Triggers Ansible

The `local-exec` provisioner in `main.tf` runs on your Terraform host (Windows PC) after each VM is created:

```hcl
provisioner "local-exec" {
  command = <<-EOT
    ssh -i ${var.ansible_key_path} \
      -o StrictHostKeyChecking=no \
      ${var.ansible_user}@192.168.198.134 \
      "ansible-playbook \
        -i '${self.default_ip_address},' \
        --private-key ${var.ansible_key_path} \
        -u ${var.ansible_user} \
        --become \
        ${var.ansible_playbook_path}"
  EOT
}
```

**What this does step by step:**

1. Terraform finishes cloning the VM and waits for an IP from `open-vm-tools`
2. `local-exec` runs an SSH command **from your Windows PC** to the Ansible control node (`192.168.198.134`)
3. On the control node, `ansible-playbook` runs against the **new VM's IP** (`self.default_ip_address`)
4. Ansible SSHes from the control node to the new VM and applies `site.yml`
5. When the playbook finishes, `terraform apply` reports success

> `self.default_ip_address` is automatically filled by Terraform with the new VM's IP — you don't hardcode it.

---

## Variables Reference

| Variable | Type | Default | Sensitive | Description |
|----------|------|---------|-----------|-------------|
| `vsphere_user` | string | — | No | vSphere username |
| `vsphere_password` | string | — | **Yes** | vSphere password (never logged) |
| `vsphere_server` | string | — | No | vCenter or ESXi host IP |
| `datacenter_name` | string | — | No | vSphere datacenter name |
| `datastore_name` | string | — | No | Datastore for VM disks |
| `network_name` | string | — | No | VM network port group |
| `esxi_host_name` | string | — | No | ESXi host IP or hostname |
| `template_name` | string | — | No | RHEL 9 template name in vSphere |
| `vm_count` | number | `1` | No | Number of VMs to create |
| `vm_prefix` | string | `rhel9-lab` | No | VM name prefix |
| `vm_cpu` | number | `2` | No | CPUs per VM |
| `vm_ram` | number | `2048` | No | RAM in MB per VM |
| `vm_disk_size` | number | `40` | No | Disk in GB per VM |
| `ansible_user` | string | `ansible_svc` | No | SSH user for Ansible |
| `ansible_key_path` | string | `/home/ansible_svc/.ssh/ansible_id_rsa` | No | SSH key path on control node |
| `ansible_playbook_path` | string | `/home/ansible_svc/ansible/rhel-lab/site.yml` | No | Playbook path on control node |

---

## Outputs

After `terraform apply`, the following are displayed:

| Output | Description | Example |
|--------|-------------|---------|
| `vm_names` | List of all created VM names | `["rhel9-lab-01", "rhel9-lab-02"]` |
| `vm_ip_addresses` | List of VM IP addresses | `["192.168.198.135", "192.168.198.136"]` |
| `vm_ids` | List of vSphere VM IDs | `["422f...", "422f..."]` |

> `vm_ip_addresses` requires `open-vm-tools` running inside the VM template.

---

## Drift Detection

To check if your vSphere VMs have drifted from your Terraform state (manual changes, vSphere auto-modifications):

```bash
terraform plan
```

If Terraform reports changes you didn't make, your infrastructure has drifted. The `lifecycle` block in `main.tf` intentionally ignores certain vSphere-managed fields:

```hcl
lifecycle {
  ignore_changes = [
    clone[0].template_uuid,    # vSphere may update this reference
    disk[0].thin_provisioned,  # vSphere normalizes this after creation
  ]
}
```

A scheduled `terraform plan` in CI/CD (Phase 5) will automate drift detection.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Error: could not find datacenter` | `datacenter_name` doesn't match vSphere exactly | Check vSphere UI — for standalone ESXi it's usually `ha-datacenter` |
| `Error: could not find template` | `template_name` mismatch | Must match exactly, including case |
| VM created but no IP in outputs | `open-vm-tools` not installed in template | `dnf install open-vm-tools -y` in template, then re-template |
| `local-exec` fails / Ansible not triggered | SSH to control node failing | Verify `ansible_key_path` is the private key path on your local machine, not the control node |
| `allow_unverified_ssl` warning | ESXi self-signed cert | Expected in lab — set `false` in production with proper cert |
| `Error: insufficient permissions` | Wrong vSphere user | Use `administrator@vsphere.local` or ensure user has VM create/clone permissions |
| VM count mismatch after apply | State drift | Run `terraform refresh` then `terraform plan` |
| Password visible in logs | Password passed via `-e` flag | Always use `TF_VAR_vsphere_password` environment variable |

---

## Security Notes

- `vsphere_password` is declared `sensitive = true` — Terraform never prints it in logs or plan output
- Always pass the password via `TF_VAR_vsphere_password` environment variable, not in `terraform.tfvars`
- `terraform.tfvars` is in `.gitignore` — never commit it with real IPs or credentials
- `allow_unverified_ssl = true` is acceptable for home lab ESXi; set to `false` in production with a valid certificate
- The `ansible_key_path` should point to a key with passphrase in production environments

---

## Related Repositories

- [ansible-sap-rhel-automation](https://github.com/<your-username>/ansible-sap-rhel-automation) — SAP ABAP + Oracle 19c deployment automation on RHEL 8

---

## License

MIT License — see [LICENSE](LICENSE) for details.

---

<p align="center">
  Part of a home lab Infrastructure as Code series — Terraform + Ansible + vSphere
</p>
