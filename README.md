![Diagramma Architettura Azure](https://raw.githubusercontent.com/Harukisan01/Azure-Architecture-Project/main/spk.png)

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FHarukisan01%2FAzure-Architecture-Project%2Fmain%2FArm.json)

# 🏗️ Guida Operativa Definitiva: Architettura Azure Hub-and-Spoke (Enterprise Grade)

Questa guida fornisce i passaggi esatti per costruire l'infrastruttura di rete, calcolo e sicurezza definita nel diagramma architetturale del progetto `Azure-Architecture-Project`, partendo da zero sul portale di Azure (https://portal.azure.com).

## 📋 PREMESSA: Il Resource Group
1. Cerca **Resource groups** e clicca su **+ Create**.
2. **Resource group**: scrivi `rg-azure-architecture`.
3. **Region**: scegli **West Europe** (o la tua region di riferimento).
4. Clicca **Review + create** e poi **Create**.

---

## 🛠️ FASE 1: Fondamenta di Rete (VNet e Subnet Avanzate)
*Nota: Il diagramma utilizza due spazi di indirizzamento separati (10.0.x.x per il frontend/integrazione e 10.1.x.x per il backend).*

1. Cerca **Virtual networks** e clicca su **+ Create**.
2. **Scheda Basics**:
   * *Resource group*: `rg-azure-architecture`.
   * *Virtual network name*: `vnet-main`.
   * *Region*: West Europe.
   * Clicca **Next: IP Addresses >**.
3. **Scheda IP Addresses**:
   * Sotto *IPv4 address space*, elimina lo spazio predefinito e aggiungi questi due:
     * `10.0.0.0/16`
     * `10.1.0.0/16`
   * Elimina la subnet `default`.
   * Clicca **+ Add a subnet** per creare le seguenti 4 sottoreti:
     * Nome: `AzureBastionSubnet` | Range: `10.0.0.0/24` -> Add.
     * Nome: `FrontEndSubnet` | Range: `10.0.1.0/24` -> Add.
     * Nome: `IntegrationSubnet` | Range: `10.0.2.0/24` -> Add.
     * Nome: `BackendSubnet` | Range: `10.1.1.0/24` -> **ATTENZIONE:** Prima di cliccare Add, scorri in basso nella finestra della subnet, espandi *Private endpoint network policies* e seleziona **Disabled** (come richiesto espressamente nel diagramma). Poi clicca Add.
4. Clicca **Review + create**, poi **Create**.

---

## 🛠️ FASE 2: Osservabilità e Backup
1. Cerca **Log Analytics workspaces** e clicca **+ Create**.
   * *Name*: `law-workspace-01`
   * Clicca **Review + create** e poi **Create**.
2. Cerca **Recovery Services vaults** e clicca **+ Create**.
   * *Vault name*: `rsv-backup-01`
   * Clicca **Review + create** e poi **Create**.
   * *(Più avanti creeremo la `DailyBackupPolicy` per proteggere le risorse).*

---

## 🛠️ FASE 3: Storage Blindato e Private DNS Zone
1. Cerca **Storage accounts** e clicca **+ Create**.
2. **Scheda Basics**:
   * *Storage account name*: nome unico globale (es. `store` + numeri).
   * *Performance*: Standard | *Redundancy*: LRS.
3. **Scheda Networking**:
   * *Network access*: seleziona **Disable public access and use private access**.
4. Clicca **Review + create** e poi **Create**. Attendi la fine dell'operazione.

**Creazione del Private Endpoint (`pe-storage`):**
5. Apri lo Storage Account appena creato -> menu a sinistra **Networking** -> scheda **Private endpoint connections** -> clicca **+ Private endpoint**.
6. **Basics**: *Name*: `pe-storage`. Next.
7. **Resource**: *Target sub-resource*: **blob**. Next.
8. **Virtual Network**: *Virtual network*: `vnet-main` | *Subnet*: **BackendSubnet** (`10.1.1.0/24`). Next.
9. **DNS**: *Integrate with private DNS zone*: **Yes**. (Questo creerà la zona `privatelink.blob.core.windows.net` e il gruppo `pe-storage/default` visti nel diagramma, collegandola alla VNet).
10. Clicca **Review + create**, poi **Create**.

---

## 🛠️ FASE 4: Frontend Web e VNet Integration
1. Cerca **App Services** e clicca **+ Create** -> **+ Web App**.
2. **Scheda Basics**:
   * *Name*: es. `app-frontend-01`.
   * *Publish*: Code | *Runtime*: (es. Node 20 LTS) | *OS*: **Linux**.
   * *Pricing plan*: Clicca "Create new", scrivi `plan-az104`. Scegli un piano Standard (es. S1).
3. **Scheda Networking**:
   * *Enable network injection*: **On**.
   * Seleziona `vnet-main` e la **IntegrationSubnet** (`10.0.2.0/24`). (Questo abilita la feature *vnetRouteAllEnabled*).
4. Clicca **Review + create** e poi **Create**.

---

## 🛠️ FASE 5: Il Bilanciatore Interno (Internal Load Balancer)
1. Cerca **Load balancers** e clicca **+ Create**.
2. **Basics**: *Name*: `lb-vmss` | *Type*: **Internal** | *SKU*: Standard. Next.
3. **Frontend IP configuration**: + Add. Nome: `ip-frontend`, VNet: `vnet-main`, Subnet: **BackendSubnet** (`10.1.1.0/24`), Dynamic. Add. Next.
4. **Backend pools**: + Add. Nome: `backend-pool-vmss`, VNet: `vnet-main`. Save. Next.
5. **Inbound rules**: + Add. 
   * Nome: `lb-rule-tcp`. Frontend: `ip-frontend`. Backend pool: `backend-pool-vmss`.
   * Protocol: **TCP**, Port **80**, Backend port **80**.
   * *Health probe*: Create new -> Nome `probe-http-5s`, Protocol **HTTP**, Port **80**, Path `/`, **Interval: 5 seconds** (Dettaglio esatto del diagramma). Salva.
6. Clicca **Review + create**, poi **Create**.
*(Nota: a creazione finita, vai nella pagina del Load Balancer e annota il suo "Private IP address", es. 10.1.1.4).*

---

## 🛠️ FASE 6: Il Motore di Calcolo (VMSS - 2 Istanze)
1. Cerca **Virtual machine scale sets** e clicca **+ Create**.
2. **Basics**: *Name*: `vmss-backend` | *Image*: Ubuntu Server 22.04 LTS. Inserisci Username/Password.
3. **Networking**: 
   * *Virtual network*: `vnet-main`.
   * Modifica la Network interface (icona matita): seleziona **BackendSubnet**.
   * Spunta **Use a load balancer**. Seleziona `lb-vmss` e `backend-pool-vmss`.
4. **Scaling**: *Initial instance count*: **2**.
5. Clicca **Review + create**, poi **Create**.

---

## 🛠️ FASE 7: Sicurezza Zero-Trust (Managed Identity & RBAC)
1. Apri l'**App Service** -> menu **Identity** -> *Status* **On** -> Save.
2. Apri il **VMSS** -> menu **Identity** -> *Status* **On** -> Save.
3. Apri lo **Storage Account** -> menu **Access Control (IAM)** -> **+ Add** -> **Add role assignment**.
4. Cerca il ruolo: **Storage Blob Data Reader**. Next.
5. *Assign access to*: **Managed identity**.
6. Clicca **+ Select members**, aggiungi sia l'App Service che il VMSS.
7. Clicca **Review + assign**.

---

## 🛠️ FASE 8: App Gateway, WAF e Routing
1. Cerca **Application Gateways** e clicca **+ Create**.
2. **Basics**: *Name*: `agw-front` | *Tier*: **WAF V2**. VNet: `vnet-main`, Subnet: **FrontEndSubnet** (`10.0.1.0/24`).
3. **Frontends**: Frontend IP type: Public. "Add new" -> `pip-agw`.
4. **Backends**:
   * Add pool 1: Nome `pool-appservice`. Target: App Services -> seleziona il tuo `app-frontend-01`.
   * Add pool 2: Nome `pool-vmss`. Target: IP address -> Inserisci l'IP del Load Balancer annotato in Fase 5 (es. `10.1.1.4`).
5. **Configuration (Routing)**:
   * Aggiungi regola: *Name* `rule-main`, *Priority* `100`.
   * *Listener*: Nome `listener-80`, Frontend IP Public, Porta **80**.
   * *Backend targets* (Default routing): Target `pool-vmss`. Crea nuovi *Backend settings* (Nome `set-80`, Port 80).
   * *Path-based rules*: Path **`/app/*`**, Target name `route-app`, Settings `set-80`, Target **`pool-appservice`**.
6. Clicca **Review + create** e poi **Create** (richiede ~15-20 min).

---

## 🛠️ FASE 9: Accesso Amministrativo (Azure Bastion)
1. Cerca **Bastion** e clicca **+ Create**.
2. *Name*: `bastion-host`. VNet: `vnet-main` (riconoscerà la `AzureBastionSubnet`).
3. *Public IP*: Create new -> `pip-bastion`.
4. Clicca **Review + create**, poi **Create**.

# Azure Architecture -- Checklist di Implementazione

## 1. Preparazione Ambiente

-   [ ] Creare Resource Group
-   [ ] Definire naming convention standard
-   [ ] Definire region di deployment
-   [ ] Configurare subscription e RBAC
-   [ ] Abilitare Microsoft Defender for Cloud (opzionale)

------------------------------------------------------------------------

## 2. Networking

### 2.1 Virtual Network

-   [ ] Creare Virtual Network (VNet)
-   [ ] Creare subnet:
    -   [ ] Subnet Application Gateway
    -   [ ] Subnet VMSS
    -   [ ] Subnet Private Endpoints
-   [ ] Configurare NSG per ogni subnet
-   [ ] Configurare UDR se necessario

### 2.2 DNS

-   [ ] Creare Private DNS Zone (privatelink.blob.core.windows.net)
-   [ ] Collegare la Private DNS Zone alla VNet
-   [ ] Verificare risoluzione DNS interna

------------------------------------------------------------------------

## 3. Accesso Sicuro

### 3.1 Azure Bastion

-   [ ] Creare subnet AzureBastionSubnet
-   [ ] Deploy Azure Bastion
-   [ ] Test accesso RDP/SSH alle VM senza IP pubblico

------------------------------------------------------------------------

## 4. Storage Layer

### 4.1 Storage Account

-   [ ] Creare Storage Account
-   [ ] Disabilitare Public Network Access
-   [ ] Impostare Access Tier (Hot/Cool)
-   [ ] Abilitare diagnosi e logging

### 4.2 Private Endpoint

-   [ ] Creare Private Endpoint per Blob
-   [ ] Associare Private DNS Zone
-   [ ] Test connettività privata

------------------------------------------------------------------------

## 5. Identity & Access

### 5.1 Managed Identity

-   [ ] Abilitare Managed Identity su:
    -   [ ] App Service
    -   [ ] VM Scale Set
-   [ ] Assegnare ruolo RBAC (es. Storage Blob Data Reader/Contributor)
-   [ ] Test autenticazione senza secret

------------------------------------------------------------------------

## 6. Compute Layer

### 6.1 App Service

-   [ ] Creare App Service Plan (Linux)
-   [ ] Creare Web App
-   [ ] Abilitare VNet Integration
-   [ ] Configurare variabili ambiente
-   [ ] Deploy applicazione

### 6.2 VM Scale Set

-   [ ] Creare VM Scale Set (min 2 istanze)
-   [ ] Configurare autoscale
-   [ ] Installare runtime applicativo
-   [ ] Abilitare diagnostica

------------------------------------------------------------------------

## 7. Load Balancing

### 7.1 Internal Load Balancer

-   [ ] Creare Internal Load Balancer
-   [ ] Configurare backend pool (VMSS)
-   [ ] Creare regola TCP (80 → 80)
-   [ ] Configurare Health Probe (HTTP)

### 7.2 Application Gateway (WAF v2)

-   [ ] Creare Public IP
-   [ ] Deploy Application Gateway (WAF v2)
-   [ ] Configurare Listener (HTTP/HTTPS)
-   [ ] Installare certificato TLS
-   [ ] Configurare backend pool
-   [ ] Configurare health probe
-   [ ] Abilitare WAF in modalità Prevention

------------------------------------------------------------------------

## 8. Monitoring

### 8.1 Log Analytics

-   [ ] Creare Log Analytics Workspace
-   [ ] Abilitare Diagnostic Settings per:
    -   [ ] Application Gateway
    -   [ ] VMSS
    -   [ ] App Service
    -   [ ] Storage Account
-   [ ] Creare alert su metriche critiche

------------------------------------------------------------------------

## 9. Backup

### 9.1 Recovery Services Vault

-   [ ] Creare Recovery Services Vault
-   [ ] Configurare Backup Policy (giornaliera)
-   [ ] Abilitare backup VM
-   [ ] Test ripristino

------------------------------------------------------------------------

## 10. Validazione Finale

-   [ ] Test accesso pubblico via Application Gateway
-   [ ] Verificare assenza IP pubblici sulle VM
-   [ ] Verificare accesso Storage solo via Private Endpoint
-   [ ] Test autoscale VMSS
-   [ ] Verificare log e monitoraggio
-   [ ] Eseguire security review finale

------------------------------------------------------------------------

## 11. Documentazione

-   [ ] Aggiornare diagramma architetturale
-   [ ] Documentare configurazioni chiave
-   [ ] Salvare export ARM/Bicep/Terraform
-   [ ] Preparare handover document



