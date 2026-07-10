# Azure Active Directory Domain Controller — Terraform Deployment Lab

Deploying a Windows Server domain controller on Azure using Terraform (Infrastructure as Code), including troubleshooting real-world deployment issues.

## Overview

This lab uses Terraform to provision a full Azure environment — resource group, virtual network, subnet, public IP, network security group, NIC, and a Windows Server VM — then uses a Custom Script Extension to automatically install Active Directory Domain Services (AD DS) and promote the server to a new forest, all from a single `terraform apply`.

**Stack:** Terraform · Azure (`azurerm` provider) · Windows Server 2022 · Active Directory Domain Services

## Architecture

```mermaid
flowchart TB
    subgraph Local["Local Machine (Windows 11)"]
        TF[Terraform CLI]
        CLI[Azure CLI]
    end

    subgraph Azure["Azure Subscription - West Europe"]
        RG[Resource Group<br/>rg-ad-veronika]
        subgraph Network
            VNET[Virtual Network<br/>10.0.0.0/16]
            SUBNET[Subnet<br/>10.0.1.0/24]
            NSG[Network Security Group<br/>Allow RDP 3389]
            PIP[Public IP<br/>Static]
        end
        NIC[Network Interface<br/>10.0.1.4]
        VM[Windows Server 2022 VM<br/>Standard_D2s_v3]
        EXT[Custom Script Extension<br/>Installs AD DS + DNS]
    end

    subgraph DC["Result: Domain Controller"]
        AD[(Active Directory<br/>corp.veronika.lab)]
        DNS[DNS Server]
    end

    TF -->|terraform apply| CLI
    CLI -->|Provisions| RG
    RG --> VNET
    VNET --> SUBNET
    SUBNET --> NIC
    NIC --> VM
    NSG --> NIC
    PIP --> NIC
    VM --> EXT
    EXT -->|Install-ADDSForest| AD
    EXT --> DNS

    User[Admin - RDP Client] -->|mstsc CORP\\adadmin| PIP
```

## Prerequisites

- Azure CLI (`az --version`)
- Terraform v1.3+ (`terraform --version`)
- An active Azure subscription with available vCPU quota
- Windows 11 laptop, regular (non-admin) PowerShell for Terraform commands

## Files

| File | Purpose |
|---|---|
| `main.tf` | All Azure resources + the AD DS install script |
| `variables.tf` | Input variable declarations |
| `terraform.tfvars` | Actual values (region, passwords, domain name) — **excluded from git via `.gitignore`** |
| `outputs.tf` | Prints public IP, domain name, and admin username after apply |

> ⚠️ `terraform.tfvars` contains plaintext passwords. It is listed in `.gitignore` and never committed. A `terraform.tfvars.example` with placeholder values is provided instead.

## Deployment Steps

```powershell
az login
cd C:\dev\az-ad-vm
terraform init
terraform plan
terraform apply
```

After apply completes, wait 5–10 minutes for the VM to finish rebooting after AD DS promotion, then RDP in:

```powershell
mstsc /v:<public_ip>
```

Login as `CORP\adadmin` (domain format, not local, since the machine has already promoted to a DC).

## Verification

Run inside the VM (PowerShell as Administrator):

```powershell
Get-Service NTDS | Select-Object Name, Status
Get-ADDomain
Get-ADDomainController -Filter *
Resolve-DnsName corp.veronika.lab
```

All four returned clean output confirming the forest, domain controller registration, and DNS resolution were working correctly.

## Issues Encountered & Fixes

This section documents the real troubleshooting done during deployment — included deliberately, since working through infrastructure failures is as much a part of the skill as the happy path.

### 1. `az login` failed with `AADSTS50076` (MFA required)
**Cause:** The browser sign-in closed before completing the multi-factor authentication step, so Azure CLI couldn't retrieve any subscriptions.
**Fix:** Re-ran `az login` and fully completed the MFA prompt in the browser before switching back to the terminal.

### 2. VM size unavailable — `SkuNotAvailable` (Standard_D2s_v3, then Standard_B2s, then Standard_B2ms)
**Cause:** Azure's free-tier/trial subscriptions get lower priority for regional VM size capacity. Multiple common sizes were temporarily out of stock in `uksouth`.
**Fix:** Discovered that West Europe, Availability Zone 3, had confirmed capacity for `Standard_D2s_v3` (validated first via a manual VM creation attempt in the Portal). Updated `location` to `westeurope` and added `zone = "3"` to the VM resource block.

### 3. Quota confusion — "0 vCPUs of 4 remain"
**Cause:** Misread as a hard blocker; actually just meant no VM was currently consuming the subscription's 4-vCPU trial limit at that moment (confirmed via `az vm list-usage` and `az vm list --show-details`).
**Fix:** No action needed once the correct VM size/region combination was found — quota was never actually the bottleneck.

### 4. `terraform.tfvars` location value in the wrong format
**Cause:** Pasted the Azure Portal's human-readable label (`"(Europe) West Europe"`) directly into `terraform.tfvars` instead of the CLI region code.
**Fix:** Changed to the correct short code: `westeurope`.

### 5. Intermittent `ResourceNotFound` / `404` errors on the Virtual Network during `apply`
**Cause:** An Azure Resource Manager timing/propagation lag — the VNet was actually being created successfully, but a near-simultaneous status check from Terraform returned a stale 404 before Azure's API had fully caught up.
**Fix:** Re-ran `terraform apply`; when Azure then reported the VNet "already exists" (since it *had* succeeded), used `terraform import` to bring the existing resource back under Terraform's management:
```powershell
terraform import azurerm_virtual_network.main /subscriptions/<sub-id>/resourceGroups/rg-ad-veronika/providers/Microsoft.Network/virtualNetworks/vnet-ad-veronika
```

### 6. Same import/apply cycle repeating — state kept "forgetting" the VNet
**Root cause (the real one):** The project folder (`C:\repos\...`) was inside a **OneDrive-synced directory**. OneDrive was periodically re-syncing `terraform.tfstate` mid-operation, silently reverting it to an older synced copy right after each successful import.
**Fix:** Moved the entire project to a non-synced local path (`C:\dev\az-ad-vm`), re-initialized Terraform there, re-imported the VNet once, and the state held correctly from then on.
**Lesson:** Never keep a Terraform working directory inside a cloud-sync folder (OneDrive, Dropbox, Google Drive) — the state file needs exclusive, uninterrupted writes.

### 7. RDP login failed — "Your credentials did not work"
**Cause:** Likely a UK/US keyboard layout mismatch affecting how special characters (e.g. `@`) were interpreted at the login screen, despite AD DS having installed correctly (confirmed via Azure's Run Command feature, which showed `NTDS: Running` and the domain user as `Enabled`/not locked out).
**Fix:** Used Azure Portal → VM → **Run command → RunPowerShellScript** to reset the domain admin password directly on the VM (bypassing RDP entirely):
```powershell
net user adadmin "<new-password>" /domain
```
Logged in successfully afterward using `CORP\adadmin` with the freshly-set password.

## Cleanup

```powershell
terraform destroy
```

Confirms with `yes`, removes every resource created (VM, disks, NIC, IP, NSG, VNet, resource group) to stop billing.

## Key Takeaways

- Infrastructure as Code doesn't eliminate cloud quirks — it surfaces them clearly and makes them reproducible/fixable.
- Regional VM capacity constraints on trial subscriptions are common and unrelated to actual quota limits — worth checking both separately.
- Cloud-synced folders (OneDrive/Dropbox) are a genuine hazard for any tool that manages its own state file.
- Azure's **Run Command** feature is a reliable way to diagnose and fix VM-level issues (like a stuck credential) without needing a working RDP session first.
