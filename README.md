![Diagramma Architettura Azure](https://raw.githubusercontent.com/Harukisan01/Azure-Architecture-Project/main/spk.png)

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FHarukisan01%2FAzure-Architecture-Project%2Fmain%2FArm.json)

# 🏗️ Definitive Operational Guide: Azure Hub-and-Spoke Architecture (Enterprise Grade)

This guide provides the exact steps to build the network, compute, and security infrastructure defined in the `Azure-Architecture-Project` architectural diagram, starting from scratch in the Azure portal (https://portal.azure.com).

---

## 📋 PREREQUISITE: Resource Group

1. Search for **Resource groups** and click **+ Create**.  
2. **Resource group**: enter `rg-azure-architecture`.  
3. **Region**: choose **West Europe** (or your preferred region).  
4. Click **Review + create**, then **Create**.

---

## 🛠️ PHASE 1: Network Foundation (VNet and Advanced Subnets)

> The diagram uses two separate address spaces:
> - `10.0.x.x` → Frontend / Integration
> - `10.1.x.x` → Backend

1. Search for **Virtual networks** and click **+ Create**.

### Basics
- Resource group: `rg-azure-architecture`
- Virtual network name: `vnet-main`
- Region: West Europe  
- Click **Next: IP Addresses**

### IP Addresses
Remove the default address space and add:

- `10.0.0.0/16`
- `10.1.0.0/16`

Delete the default subnet.

Create the following subnets:

| Subnet Name          | Address Range    | Notes |
|----------------------|------------------|-------|
| AzureBastionSubnet  | 10.0.0.0/24      | Required for Bastion |
| FrontEndSubnet      | 10.0.1.0/24      | Application Gateway |
| IntegrationSubnet   | 10.0.2.0/24      | App Service VNet Integration |
| BackendSubnet       | 10.1.1.0/24      | Disable Private Endpoint Policies |

⚠ For `BackendSubnet`, disable **Private endpoint network policies** before saving.

Click **Review + create** → **Create**.

---

## 🛠️ PHASE 2: Observability and Backup

### Log Analytics Workspace

- Name: `law-workspace-01`
- Review + create → Create

### Recovery Services Vault

- Name: `rsv-backup-01`
- Review + create → Create

---

## 🛠️ PHASE 3: Hardened Storage + Private Endpoint

### Storage Account

- Name: globally unique (e.g., `store12345`)
- Performance: Standard
- Redundancy: LRS
- Networking: **Disable public access**

Deploy.

---

### Private Endpoint (`pe-storage`)

- Target sub-resource: **blob**
- VNet: `vnet-main`
- Subnet: `BackendSubnet`
- Integrate with Private DNS zone: **Yes**

This creates and links:

`privatelink.blob.core.windows.net`

---

## 🛠️ PHASE 4: Web Frontend + VNet Integration

### App Service

- Name: `app-frontend-01`
- Runtime: Node 20 LTS (example)
- OS: Linux
- Plan: `plan-az104` (S1)

### Networking

- Enable network injection: **On**
- VNet: `vnet-main`
- Subnet: `IntegrationSubnet`

Deploy.

---

## 🛠️ PHASE 5: Internal Load Balancer

- Name: `lb-vmss`
- Type: Internal
- SKU: Standard
- Frontend IP: BackendSubnet
- Backend Pool: `backend-pool-vmss`

### Rule

- Protocol: TCP
- Port: 80 → 80

### Health Probe

- HTTP
- Port: 80
- Path: `/`
- Interval: 5 seconds

Note the private IP assigned.

---

## 🛠️ PHASE 6: VM Scale Set (2 Instances)

- Name: `vmss-backend`
- Image: Ubuntu 22.04 LTS
- VNet: `vnet-main`
- Subnet: `BackendSubnet`
- Attach to Load Balancer
- Instance count: 2

Deploy.

---

## 🛠️ PHASE 7: Managed Identity + RBAC

Enable **System Assigned Identity** on:

- App Service
- VM Scale Set

Assign RBAC role on Storage:

- Role: `Storage Blob Data Reader`
- Assign to: Managed Identities

---

## 🛠️ PHASE 8: Application Gateway (WAF v2)

- Name: `agw-front`
- Tier: WAF V2
- Subnet: `FrontEndSubnet`
- Public IP: `pip-agw`

### Backend Pools

- `pool-appservice` → App Service
- `pool-vmss` → Internal LB IP

### Routing Rule

- Listener: Port 80
- Default: VMSS
- Path `/app/*` → App Service

Deploy (~20 min).

---

## 🛠️ PHASE 9: Azure Bastion

- Name: `bastion-host`
- Subnet: `AzureBastionSubnet`
- Public IP: `pip-bastion`

Deploy.

---

# ✅ Implementation Checklist

## Environment

- [ ] Resource Group created
- [ ] Naming convention defined
- [ ] Region defined
- [ ] RBAC configured
- [ ] Defender for Cloud enabled (optional)

---

## Networking

- [ ] VNet created
- [ ] Subnets configured
- [ ] NSGs configured
- [ ] UDRs configured (if needed)

---

## DNS

- [ ] Private DNS Zone created
- [ ] Linked to VNet
- [ ] Internal resolution verified

---

## Secure Access

- [ ] Bastion deployed
- [ ] No public IPs on VMs
- [ ] SSH/RDP tested via Bastion

---

## Storage

- [ ] Public access disabled
- [ ] Private Endpoint configured
- [ ] DNS integration verified

---

## Identity

- [ ] Managed Identities enabled
- [ ] RBAC assigned
- [ ] Secretless authentication tested

---

## Compute

- [ ] App Service deployed
- [ ] VNet Integration enabled
- [ ] VMSS deployed (2 instances)
- [ ] Autoscale configured

---

## Load Balancing

- [ ] Internal Load Balancer working
- [ ] Health probe responding
- [ ] Application Gateway deployed
- [ ] WAF in Prevention mode
- [ ] TLS configured (if HTTPS enabled)

---

## Monitoring

- [ ] Log Analytics connected
- [ ] Diagnostic settings enabled
- [ ] Alerts configured

---

## Backup

- [ ] Backup policy created
- [ ] VM backup enabled
- [ ] Restore test completed

---

## Final Validation

- [ ] Public access via Application Gateway works
- [ ] Storage accessible only via Private Endpoint
- [ ] Autoscale tested
- [ ] Logs verified
- [ ] Security review completed

---

## Documentation

- [ ] Architecture diagram updated
- [ ] Key configurations documented
- [ ] ARM/Bicep/Terraform exported
- [ ] Handover documentation prepared
