# Azure Active Directory Domain Controller — Terraform Deployment

## WATCH ME BUILD IT --> https://canva.link/lw1ipeqlr0uz8hu

**Infrastructure-as-Code lab:** provisioning a Windows Server 2022 Active Directory Domain Controller on Azure using Terraform, with a fully automated AD DS installation and forest promotion.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Deployment](#deployment)
- [Verification](#verification)
- [Troubleshooting Log](#troubleshooting-log)
- [Cleanup](#cleanup)
- [Skills Demonstrated](#skills-demonstrated)
- [Author](#author)

---

## Overview

This project deploys a complete Active Directory environment on Azure entirely through Terraform — no manual clicking through the Azure Portal for the infrastructure, and no manual Server Manager wizards for the AD role.

A single `terraform apply` provisions:

- A dedicated resource group, virtual network, and subnet
- A network security group scoped to allow RDP
- A static public IP and network interface
- A Windows Server 2022 virtual machine
- A Custom Script Extension that installs the AD DS role and promotes the server to a new Active Directory forest, including DNS

The result is a fully functional, reproducible domain controller that can be built, verified, and torn down on demand — a pattern directly applicable to real enterprise environments where infrastructure is version-controlled rather than manually configured.

---

## Architecture

```mermaid
flowchart TB
    subgraph Local["Local Workstation (Windows 11)"]
        TF[Terraform CLI]
        CLI[Azure CLI]
    end

    subgraph Azure["Azure Subscription — West Europe"]
        RG[("Resource Group<br/>rg-ad-veronika")]
        subgraph Network["Networking"]
            VNET["Virtual Network<br/>10.0.0.0/16"]
            SUBNET["Subnet<br/>10.0.1.0/24"]
            NSG["Network Security Group<br/>Allow RDP :3389"]
            PIP["Public IP<br/>Static"]
        end
        NIC["Network Interface<br/>10.0.1.4"]
        VM["Windows Server 2022 VM<br/>Standard_D2s_v3"]
        EXT["Custom Script Extension<br/>Install-ADDSForest"]
    end

    subgraph Result["Provisioned Domain"]
        AD[("Active Directory<br/>corp.veronika.lab")]
        DNS["DNS Server"]
    end

    Admin(["Administrator"]) -->|az login, terraform apply| TF
    TF --> CLI
    CLI --> RG
    RG --> VNET --> SUBNET --> NIC
    NSG --> NIC
    PIP --> NIC
    NIC --> VM --> EXT
    EXT --> AD
    EXT --> DNS
    Admin -->|RDP: CORP\\adadmin| PIP
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Provisioning | Terraform (`hashicorp/azurerm` provider) |
| Cloud Platform | Microsoft Azure |
| Operating System | Windows Server 2022 Datacenter |
| Directory Service | Active Directory Domain Services (AD DS) + DNS |
| Automation | Azure Custom Script Extension (PowerShell) |
| CLI Tooling | Azure CLI, PowerShell |

---

## Prerequisites

- [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) installed and authenticated (`az login`)
- [Terraform](https://developer.hashicorp.com/terraform/install) v1.3 or later
- An active Azure subscription with available compute quota
- Windows PowerShell (non-admin is sufficient for all Terraform commands)

---

## Repository Structure

```
az-ad-vm/
├── main.tf                    # Core resource definitions + AD DS install script
├── variables.tf                # Input variable declarations
├── outputs.tf                  # Public IP, domain name, admin username outputs
├── terraform.tfvars.example    # Sample values — copy to terraform.tfvars and edit
├── .gitignore                  # Excludes terraform.tfvars, .terraform/, state files
├── screenshots/                # Deployment evidence referenced below
└── README.md
```

> **Security note:** `terraform.tfvars` contains plaintext credentials and is excluded from version control via `.gitignore`. Only `terraform.tfvars.example` (placeholder values) is committed.

---

## Deployment

```powershell
# Authenticate
az login

# Initialize providers
terraform init

# Review the execution plan
terraform plan

# Provision the environment
terraform apply
```

Apply takes roughly 8–13 minutes total: VM provisioning (~5–8 min), followed by the AD DS installation and automatic reboot (~3–5 min).

**Clean `terraform plan` output — 8 resources queued, no errors:**

 <img width="536" height="234" alt="01-terraform-plan" src="https://github.com/user-attachments/assets/dce4280e-0eaa-4ba9-832e-826432a98cc8" />

**Successful `terraform apply` — resources created and outputs printed:**

<img width="464" height="288" alt="Screenshot 2026-07-18 232957" src="https://github.com/user-attachments/assets/9b7c57b6-281b-4491-9f87-02bab1db52ea" />

Once complete, Terraform prints the public IP, domain name, and admin username as outputs. Connect via RDP:

```powershell
mstsc /v:<public_ip>
```

**Note:** authenticate using the domain-qualified account (`CORP\adadmin` or `adadmin@corp.veronika.lab`), not a local account — the server has already been promoted to a domain controller by the time it's reachable.

---

## Verification

Run inside the VM, PowerShell as Administrator:

```powershell
Get-Service NTDS | Select-Object Name, Status
Get-ADDomain
Get-ADDomainController -Filter *
Resolve-DnsName corp.veronika.lab
```

All four commands returned clean output with no errors, confirming:
- The NTDS service (core AD engine) is running
- The forest and domain are correctly configured
- The domain controller is registered
- DNS is resolving the domain correctly

---

## Troubleshooting Log

Real issues encountered during deployment, along with root cause and resolution. Included deliberately — diagnosing infrastructure failures is as core to this skill set as a clean deployment.

### 1. `az login` failed with `AADSTS50076` (MFA required)
**Cause:** The MFA prompt wasn't completed before the browser session closed, so Azure CLI couldn't retrieve any subscriptions.
**Fix:** Re-ran `az login`, completed MFA fully, confirmed with `az account show`.

### 2. VM size unavailable — `SkuNotAvailable`
**Cause:** Regional capacity exhaustion — `Standard_D2s_v3`, then `Standard_B2s`, then `Standard_B2ms` were all temporarily out of stock in `uksouth` (common on trial subscriptions, which get lower capacity priority).

<img width="1802" height="1642" alt="03-sku-not-available" src="https://github.com/user-attachments/assets/3ece2b68-b69a-47fa-bb2d-da240831d3a8" />


**Fix:** Verified available sizes manually via the Azure Portal's VM size picker, then moved the deployment to **West Europe, Availability Zone 3**, which had confirmed capacity for `Standard_D2s_v3`.

<img width="1848" height="2292" alt="04-vm-size-picker" src="https://github.com/user-attachments/assets/27e60339-72d0-42e4-8d33-78d4a3e963fa" />


### 3. Quota confusion — "0 vCPUs of 4 remain"
**Cause:** Initially looked like a hard blocker; `az vm list-usage` confirmed 0 vCPUs were actually in use at the time. The warning referred to total subscription capacity, not something already consuming it.
**Fix:** No remediation needed — resolved automatically once the correct size/region/zone combination was in place (see #2).

### 4. `terraform.tfvars` location value in the wrong format
**Cause:** Pasted the Azure Portal's human-readable label (`"(Europe) West Europe"`) directly into `terraform.tfvars` instead of the CLI region code.
**Fix:** Corrected to `location = "westeurope"`, verified with `Select-String -Path terraform.tfvars -Pattern "location"`.

### 5. Intermittent `404 ResourceNotFound` on the Virtual Network mid-apply

<img width="1755" height="394" alt="Screenshot 2026-07-10 040433" src="https://github.com/user-attachments/assets/d5b8ec01-fc75-415d-aeff-4dc2730e25a3" />


**Cause:** Azure Resource Manager propagation lag — the VNet was actually created successfully, but a near-simultaneous status check from Terraform hit a stale 404 before Azure's API had caught up.
**Fix:** Re-ran `terraform apply`. Azure then correctly reported the VNet "already exists," so `terraform import` was used to reconcile Terraform's state with the resource that had, in fact, been created:

<img width="1639" height="1351" alt="06-terraform-import-success" src="https://github.com/user-attachments/assets/a773b09d-484c-4233-b401-8ac58abab187" />




```powershell
terraform import azurerm_virtual_network.main /subscriptions/<sub-id>/resourceGroups/rg-ad-veronika/providers/Microsoft.Network/virtualNetworks/vnet-ad-veronika
```

### 6. Same import/apply cycle repeating — state kept "forgetting" the VNet
**Root cause:** The project folder was inside a **OneDrive-synced directory**, and OneDrive sync was still active. OneDrive was periodically re-syncing `terraform.tfstate` in the background, silently reverting it to an older synced copy immediately after each successful import wrote the new state.
**Fix:** Paused OneDrive sync for the folder (no relocation needed). Re-ran the import once more, and the state held correctly on every subsequent `apply`:

<img width="1762" height="446" alt="Screenshot 2026-07-10 040749" src="https://github.com/user-attachments/assets/1db8250c-7239-4394-be78-b794067b8905" />


**Lesson:** A Terraform working directory doesn't need to live outside a cloud-sync folder permanently, but sync must be paused for that folder while Terraform is actively writing to `terraform.tfstate` — otherwise the sync client can race with Terraform and silently revert the state file.

### 7. RDP login failed — "Your credentials did not work"

<img width="1848" height="2341" alt="08-rdp-credentials-failed" src="https://github.com/user-attachments/assets/5627341f-4eee-409b-993d-27fa7520a161" />



**Diagnosis:** Used Azure Portal → VM → **Run command → RunPowerShellScript** to confirm the domain controller itself was healthy without needing a working RDP session:

```powershell
Get-Service NTDS -ErrorAction SilentlyContinue | Select Name, Status
Get-WindowsFeature AD-Domain-Services | Select Name, InstallState
```

<img width="1848" height="2391" alt="09-run-command-ntds-status" src="https://github.com/user-attachments/assets/26395f78-c16f-4511-b46a-da98e2836de2" />



```powershell
Get-ADUser -Identity adadmin -Properties LockedOut, Enabled
```
<img width="1848" height="2241" alt="10-run-command-aduser-check" src="https://github.com/user-attachments/assets/94e1e9ac-2972-47d3-8efa-04ff16b99329" />


This confirmed `NTDS: Running`, AD DS installed, and the `adadmin` account `Enabled: True`, `LockedOut: False` — ruling out a broken domain and isolating the issue to the credential itself (likely a UK/US keyboard layout mismatch affecting special characters like `@`).

**Fix:** Used Run Command to reset the domain admin password directly on the VM, bypassing RDP entirely:

```powershell
net user adadmin "<new-password>" /domain
```

Logged in successfully afterward using `CORP\adadmin` with the freshly-set password.

### Key Lessons

- **Regional VM capacity ≠ subscription quota.** These are two independent constraints and need to be diagnosed separately.
- **Cloud-synced folders (OneDrive, Dropbox, Google Drive) are a real hazard for stateful tools.** Any tool that manages its own state file — Terraform included — needs exclusive, uninterrupted write access while it's running.
- **Azure's Run Command feature is a powerful diagnostic tool.** It allowed verifying AD DS health and resetting a domain password entirely through the Azure control plane, without needing a working RDP session first.

---

## Cleanup

```powershell
terraform destroy
```

Removes every resource created — VM, disks, NIC, public IP, NSG, VNet, and resource group — in a single confirmed operation, stopping any further billing.

---

## Skills Demonstrated

- Infrastructure as Code (Terraform: providers, resources, variables, outputs, state management)
- Azure networking fundamentals (VNets, subnets, NSGs, public IPs)
- Windows Server administration and Active Directory Domain Services deployment
- PowerShell scripting and remote diagnostics via Azure Run Command
- Systematic troubleshooting: isolating root cause across CLI auth, cloud capacity, state management, and credential issues
- Documentation of a real deployment process, including failures and their resolutions

---

## Author

**Veronika**
Cybersecurity portfolio project — built as part of hands-on Azure home lab work.
