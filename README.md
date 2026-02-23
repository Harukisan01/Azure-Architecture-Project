# Azure-Architecture-Project


Ecco la guida completa e unificata, formattata in Markdown.

Puoi copiare l'intero blocco di testo qui sotto, incollarlo in un editor di testo (come Blocco Note, VS Code o Notepad++) e salvarlo con il nome Guida-Infrastruttura-Azure.md.

code
Markdown
download
content_copy
expand_less
# 🏗️ Guida Operativa: Implementazione Architettura Azure Hub-and-Spoke

Questa guida fornisce i passaggi esatti, "clic-per-clic", per costruire l'infrastruttura di rete, calcolo e sicurezza definita nel diagramma architetturale, partendo da zero sul portale di Azure (https://portal.azure.com).

## 📋 Prerequisito: Il Contenitore (Resource Group)
Tutte le risorse in Azure devono risiedere in un gruppo logico.
1. Nella barra di ricerca in alto, scrivi **Resource groups** e clicca sul servizio.
2. Clicca su **+ Create**.
3. **Resource group**: scrivi `RG-Architettura`.
4. **Region**: scegli **West Europe** (o la regione desiderata).
5. Clicca **Review + create**, poi **Create**.

---

## 🛠️ PASSO 1: Costruire la Rete e le Sottoreti (Virtual Network)
Creiamo il perimetro isolato per le nostre risorse.
1. Cerca in alto **Virtual networks** e clicca su **+ Create**.
2. **Scheda Basics**:
   * *Resource group*: `RG-Architettura`.
   * *Virtual network name*: `vnet-main`.
   * *Region*: West Europe.
   * Clicca **Next: IP Addresses >**.
3. **Scheda IP Addresses**:
   * Lascia lo spazio indirizzi predefinito (es. `10.0.0.0/16`).
   * Elimina la subnet "default" cliccando sul cestino.
   * Clicca **+ Add a subnet** per creare le seguenti 4 sottoreti (cliccando "Add" per ognuna):
     * Nome: `FrontEndSubnet` | Range: `10.0.1.0/24`
     * Nome: `IntegrationSubnet` | Range: `10.0.2.0/24`
     * Nome: `BackendSubnet` | Range: `10.0.3.0/24`
     * Nome: `AzureBastionSubnet` *(scritto esattamente così)* | Range: `10.0.4.0/24`
4. Clicca **Review + create**, poi **Create**.

---

## 🛠️ PASSO 2: Il Monitoraggio Centralizzato (Log Analytics)
Prepariamo il "raccoglitore" dei log per tutta l'infrastruttura.
1. Cerca in alto **Log Analytics workspaces** e clicca **+ Create**.
2. **Scheda Basics**:
   * *Resource group*: `RG-Architettura`.
   * *Name*: `law-miaapp-prod`.
   * *Region*: West Europe.
3. Clicca **Review + create**, poi **Create**.

---

## 🛠️ PASSO 3: Lo Storage Blindato (Storage Account & Private Link)
Creiamo lo storage e disabilitiamo l'accesso da Internet per massima sicurezza.
1. Cerca **Storage accounts** e clicca **+ Create**.
2. **Scheda Basics**:
   * *Resource group*: `RG-Architettura`.
   * *Storage account name*: nome unico, tutto minuscolo (es. `storageprivatomario88`).
   * *Region*: West Europe.
   * *Performance*: Standard.
   * *Redundancy*: Locally-redundant storage (LRS).
3. Vai alla **Scheda Networking**:
   * Sotto *Network access*, seleziona **Disable public access and use private access**.
4. Clicca **Review + create**, poi **Create**. Attendi la creazione.

**Creazione del Tunnel Privato (Private Endpoint):**
5. Apri lo Storage Account appena creato.
6. Nel menu a sinistra, sotto *Security + networking*, clicca **Networking**.
7. Vai nella scheda **Private endpoint connections** in alto e clicca **+ Private endpoint**.
8. **Scheda Basics**: *Name* `pe-storage`, vai su *Next*.
9. **Scheda Resource**: *Target sub-resource* scegli **blob**, vai su *Next*.
10. **Scheda Virtual Network**: seleziona `vnet-main` e la subnet **BackendSubnet**, vai su *Next*.
11. **Scheda DNS**: *Integrate with private DNS zone* su **Yes**.
12. Clicca **Review + create**, poi **Create**.

---

## 🛠️ PASSO 4: Il Frontend Web (App Service con VNet Integration)
1. Cerca **App Services** e clicca **+ Create** -> **+ Web App**.
2. **Scheda Basics**:
   * *Resource group*: `RG-Architettura`.
   * *Name*: nome unico (es. `app-frontend-mario88`).
   * *Publish*: Code.
   * *Runtime stack*: (es. Node 20 LTS o .NET).
   * *Operating System*: **Linux**.
   * *Pricing plan*: Clicca "Create new", scrivi `plan-az104`. Scegli un piano Standard (es. S1) (Non usare Free F1/D1).
3. Vai alla **Scheda Networking**:
   * Sotto *Enable network injection*, metti su **On**.
   * Seleziona `vnet-main` e la subnet **IntegrationSubnet**.
4. Clicca **Review + create**, poi **Create**.

---

## 🛠️ PASSO 5: Il Bilanciatore Interno (Internal Load Balancer)
Smista il traffico verso i server backend, senza IP pubblico.
1. Cerca **Load balancers** e clicca **+ Create**.
2. **Scheda Basics**:
   * *Name*: `lb-vmss` | *Type*: **Internal** | *SKU*: Standard.
3. **Scheda Frontend IP configuration**:
   * Clicca **+ Add**. Nome: `ip-frontend-interno`, VNet: `vnet-main`, Subnet: **BackendSubnet**, Assignment: Dynamic. Clicca Add.
4. **Scheda Backend pools**:
   * Clicca **+ Add**. Nome: `backend-pool-vmss`, VNet: `vnet-main`. Clicca Save.
5. **Scheda Inbound rules**:
   * Clicca **+ Add**. Nome: `regola-http-80`. Frontend: `ip-frontend-interno`. Backend pool: `backend-pool-vmss`. Protocol: TCP, Port 80, Backend port 80.
   * *Health probe*: Create new -> Nome `health-probe-80`, HTTP, Porta 80, Path `/`. Salva.
6. Clicca **Review + create**, poi **Create**.
*(Nota: a creazione finita, annota l'Indirizzo IP Privato assegnato al Load Balancer)*.

---

## 🛠️ PASSO 6: Il Motore di Calcolo (Virtual Machine Scale Set - VMSS)
1. Cerca **Virtual machine scale sets** e clicca **+ Create**.
2. **Scheda Basics**:
   * *Name*: `vmss-backend` | *Image*: Ubuntu Server 22.04 LTS (o Windows).
   * Inserisci Username e Password per l'amministratore.
3. Vai alla **Scheda Networking**:
   * Seleziona `vnet-main`. Modifica (icona matita) la Network interface e scegli la **BackendSubnet**.
   * Spunta **Use a load balancer**. Seleziona `lb-vmss` e `backend-pool-vmss`.
4. Vai alla **Scheda Scaling**:
   * *Initial instance count*: **2**.
5. Clicca **Review + create**, poi **Create**.

---

## 🛠️ PASSO 7: Sicurezza Senza Password (Managed Identity & RBAC)
Autorizziamo App Service e VMSS a leggere lo Storage senza stringhe di connessione.
1. Apri l'**App Service** -> menu a sinistra **Identity** -> *Status* su **On** -> Save.
2. Apri il **VMSS** -> menu a sinistra **Identity** -> *Status* su **On** -> Save.
3. Apri lo **Storage Account** -> menu a sinistra **Access Control (IAM)**.
4. Clicca **+ Add** -> **Add role assignment**.
5. Cerca e seleziona il ruolo **Storage Blob Data Reader**, vai su Next.
6. Assegna l'accesso a **Managed identity**.
7. Clicca **+ Select members**, scegli l'App Service e il VMSS creati.
8. Clicca **Review + assign**.

---

## 🛠️ PASSO 8: L'Ingresso su Internet (Application Gateway & WAF)
Il proxy pubblico che protegge dagli attacchi e smista le rotte.
1. Cerca **Application Gateways** e clicca **+ Create**.
2. **Scheda Basics**:
   * *Name*: `agw-front` | *Tier*: **WAF V2**.
   * Seleziona `vnet-main` e la **FrontEndSubnet**.
3. **Scheda Frontends**:
   * Frontend IP type: Public. Clicca "Add new" e chiamalo `pip-agw`.
4. **Scheda Backends**:
   * Add pool 1: Nome `pool-appservice`. Target type: App Services. Seleziona il tuo App Service.
   * Add pool 2: Nome `pool-vmss-interno`. Target type: IP address. Inserisci l'IP privato del Load Balancer (dal Passo 5).
5. **Scheda Configuration (Routing Rules)**:
   * Aggiungi regola: *Name* `regola-smistamento`, *Priority* `100`.
   * *Listener*: Nome `listener-80`, Frontend IP Public, Porta 80.
   * *Backend targets* (Default): Target `pool-vmss-interno`. *Backend settings* -> Add new (Nome `impostazioni-80`, Port 80).
   * *Path-based rules* (per /app/*): Path `/app/*`, Target name `traffico-app-service`, Settings `impostazioni-80`, Target `pool-appservice`.
6. Clicca **Review + create**, poi **Create**. *(L'operazione richiede circa 15-20 minuti).*

---

## 🛠️ PASSO 9: Gestione Sicura e Backup (Bastion & RSV)
**Azure Bastion (Accesso remoto sicuro via Browser):**
1. Cerca **Bastion** e clicca **+ Create**.
2. *Name*: `bastion-host`. Seleziona `vnet-main` (riconoscerà automaticamente la `AzureBastionSubnet`).
3. *Public IP*: Create new -> `pip-bastion`.
4. Clicca **Review + create**, poi **Create**.

**Recovery Services Vault (Backup):**
1. Cerca **Recovery Services vaults** e clicca **+ Create**.
2. *Vault name*: `rsv-backup`.
3. Clicca **Review + create**, poi **Create**.
4. Nel vault, vai su **Backup** -> Azure -> Virtual Machine, e seleziona il tuo VMSS per applicare la `DailyBackupPolicy`.

---
🎉 **INFRASTRUTTURA COMPLETATA!**
Hai implementato con successo l'architettura sicura Hub-and-Spoke.
