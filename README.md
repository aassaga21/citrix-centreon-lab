# TP — Supervision d'une infrastructure Citrix VDI avec Centreon sur Infomaniak OpenStack

> **Auteur :** Alexandra ASSAGA  
> **GitHub :** [aassaga21](https://github.com/aassaga21)  
> **Date :** Juillet 2026  
> **Durée estimée :** 2–3 jours  
> **Tags :** `Terraform` `Citrix CVAD` `Centreon` `Infomaniak` `OpenStack` `Supervision` `VDI`

---

## Objectifs pédagogiques

- Déployer une infrastructure VDI Citrix CVAD via **Terraform** sur Infomaniak Public Cloud (OpenStack)
- Superviser les composants Citrix avec **Centreon** (checks système + checks métier via PowerShell)
- Maîtriser le cycle complet **provision → configurer → superviser → détruire** (`terraform destroy`)
- Créer des plugins de supervision Nagios/Centreon personnalisés pour Citrix

---

## Architecture cible

```
┌─────────────────────────────────────────────────────────────────┐
│                  Infomaniak Public Cloud (OpenStack)            │
│                  Réseau : 192.168.100.0/24                      │
│                                                                 │
│  ┌──────────────────┐    ┌──────────────────┐                  │
│  │  vm-ddc          │    │  vm-storefront   │                  │
│  │  Delivery        │◄──►│  StoreFront      │                  │
│  │  Controller      │    │  (portail web)   │                  │
│  │  192.168.100.10  │    │  192.168.100.11  │                  │
│  └────────┬─────────┘    └──────────────────┘                  │
│           │                                                     │
│  ┌────────▼─────────┐    ┌──────────────────┐                  │
│  │  vm-sql          │    │  vm-vda          │                  │
│  │  SQL Server 2022 │    │  Session Host    │                  │
│  │  (BDD Citrix)    │    │  (Windows + VDA) │                  │
│  │  192.168.100.12  │    │  192.168.100.13  │                  │
│  └──────────────────┘    └──────────────────┘                  │
│                                                                 │
│  ┌──────────────────┐                                           │
│  │  vm-centreon     │  ← supervise tout                        │
│  │  AlmaLinux 9     │    via NRPE / SNMP / SSH                 │
│  │  192.168.100.20  │                                           │
│  └──────────────────┘                                           │
└─────────────────────────────────────────────────────────────────┘
```

| VM | Rôle | OS | Flavor |
|---|---|---|---|
| vm-ddc | Citrix Delivery Controller | Windows Server 2022 | a8-ram16-disk100-perf1 |
| vm-storefront | Citrix StoreFront | Windows Server 2022 | a4-ram8-disk60-perf1 |
| vm-sql | SQL Server 2022 Express | Windows Server 2022 | a4-ram8-disk80-perf1 |
| vm-vda | Virtual Delivery Agent (session host) | Windows Server 2022 | a4-ram8-disk60-perf1 |
| vm-centreon | Centreon Central | AlmaLinux 9 | a2-ram4-disk50-perf1 |

---

## Prérequis

### Côté Infomaniak

- Compte Infomaniak Public Cloud actif avec quota suffisant (5 VMs, ~350 Go disk)
- **Image Windows Server 2022** uploadée dans OpenStack (Glance) — voir section ci-dessous
- Image **AlmaLinux 9** disponible dans le catalogue Infomaniak (déjà présente par défaut)
- Paire de clés SSH créée dans le projet OpenStack

### Côté poste local

```bash
# Vérifier les outils
terraform --version   # >= 1.7
openstack --version   # CLI OpenStack
python3 --version
```

### Upload de l'image Windows Server 2022

> Infomaniak ne fournit pas d'image Windows par défaut. Il faut uploader un ISO converti en `.qcow2` ou utiliser une image QCOW2 communautaire.

```bash
# Télécharger une image Windows Server 2022 eval (qcow2)
# Source : https://cloudbase.it/windows-cloud-images/ (Cloudbase-Init intégré)
wget https://cloudbase.it/downloads/windows-server-2022.qcow2.xz
xz -d windows-server-2022.qcow2.xz

# Upload vers Glance Infomaniak
openstack image create "Windows-Server-2022" \
  --file windows-server-2022.qcow2 \
  --disk-format qcow2 \
  --container-format bare \
  --property os_type=windows \
  --property os_distro=windows \
  --public
```

### Récupérer les credentials OpenStack

```bash
# Depuis le dashboard Infomaniak > API Access > Télécharger openrc.sh
source ~/infomaniak-openrc.sh

# Vérifier
openstack token issue
```

---

## Structure du projet Terraform

```
citrix-centreon-lab/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars          # ← ne pas committer (credentials)
├── network.tf
├── security_groups.tf
├── vms/
│   ├── vm-ddc.tf
│   ├── vm-storefront.tf
│   ├── vm-sql.tf
│   ├── vm-vda.tf
│   └── vm-centreon.tf
└── scripts/
    ├── cloud-init-windows.ps1
    └── cloud-init-centreon.sh
```

---

## Terraform — Code complet

### `variables.tf`

```hcl
variable "os_auth_url" {
  description = "URL d'authentification OpenStack Infomaniak"
  type        = string
}

variable "os_user_name" {
  description = "Nom d'utilisateur OpenStack"
  type        = string
}

variable "os_password" {
  description = "Mot de passe OpenStack"
  type        = string
  sensitive   = true
}

variable "os_tenant_name" {
  description = "Nom du projet OpenStack"
  type        = string
}

variable "os_region_name" {
  description = "Région Infomaniak"
  type        = string
  default     = "dc3-a"
}

variable "keypair_name" {
  description = "Nom de la paire de clés SSH dans OpenStack"
  type        = string
  default     = "alex-lab-key"
}

variable "windows_image" {
  description = "Nom de l'image Windows Server 2022 dans Glance"
  type        = string
  default     = "Windows-Server-2022"
}

variable "linux_image" {
  description = "Image AlmaLinux 9 pour Centreon"
  type        = string
  default     = "AlmaLinux 9"
}

variable "admin_password" {
  description = "Mot de passe administrateur Windows"
  type        = string
  sensitive   = true
  default     = "P@ssword123!"
}

variable "network_cidr" {
  type    = string
  default = "192.168.100.0/24"
}
```

### `terraform.tfvars` (exemple — à adapter)

```hcl
os_auth_url    = "https://api.pub1.infomaniak.cloud/identity"
os_user_name   = "PCU-XXXXX"
os_password    = "TON_MOT_DE_PASSE"
os_tenant_name = "TON_PROJET"
keypair_name   = "alex-lab-key"
admin_password = "Citrix@Lab2026!"
```

### `main.tf`

```hcl
terraform {
  required_version = ">= 1.7"
  required_providers {
    openstack = {
      source  = "terraform-provider-openstack/openstack"
      version = "~> 2.1"
    }
  }
}

provider "openstack" {
  auth_url    = var.os_auth_url
  user_name   = var.os_user_name
  password    = var.os_password
  tenant_name = var.os_tenant_name
  region      = var.os_region_name
}
```

### `network.tf`

```hcl
# Réseau privé du lab
resource "openstack_networking_network_v2" "citrix_net" {
  name           = "citrix-lab-network"
  admin_state_up = true
}

resource "openstack_networking_subnet_v2" "citrix_subnet" {
  name            = "citrix-lab-subnet"
  network_id      = openstack_networking_network_v2.citrix_net.id
  cidr            = var.network_cidr
  ip_version      = 4
  dns_nameservers = ["8.8.8.8", "8.8.4.4"]

  allocation_pool {
    start = "192.168.100.10"
    end   = "192.168.100.100"
  }
}

# Router vers l'extérieur
data "openstack_networking_network_v2" "ext_net" {
  name = "ext-net1"
}

resource "openstack_networking_router_v2" "citrix_router" {
  name                = "citrix-lab-router"
  admin_state_up      = true
  external_network_id = data.openstack_networking_network_v2.ext_net.id
}

resource "openstack_networking_router_interface_v2" "citrix_router_iface" {
  router_id = openstack_networking_router_v2.citrix_router.id
  subnet_id = openstack_networking_subnet_v2.citrix_subnet.id
}

# Floating IPs pour accès externe
resource "openstack_networking_floatingip_v2" "fip_ddc" {
  pool = "ext-net1"
}

resource "openstack_networking_floatingip_v2" "fip_centreon" {
  pool = "ext-net1"
}

resource "openstack_networking_floatingip_association_v2" "fip_ddc_assoc" {
  floating_ip = openstack_networking_floatingip_v2.fip_ddc.address
  port_id     = openstack_compute_instance_v2.vm_ddc.network[0].port
}

resource "openstack_networking_floatingip_association_v2" "fip_centreon_assoc" {
  floating_ip = openstack_networking_floatingip_v2.fip_centreon.address
  port_id     = openstack_compute_instance_v2.vm_centreon.network[0].port
}
```

### `security_groups.tf`

```hcl
# SG Windows (Citrix)
resource "openstack_networking_secgroup_v2" "sg_citrix" {
  name        = "sg-citrix-windows"
  description = "Security group pour les VMs Citrix Windows"
}

resource "openstack_networking_secgroup_rule_v2" "rdp" {
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = 3389
  port_range_max    = 3389
  remote_ip_prefix  = "0.0.0.0/0"
  security_group_id = openstack_networking_secgroup_v2.sg_citrix.id
}

resource "openstack_networking_secgroup_rule_v2" "winrm" {
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = 5985
  port_range_max    = 5986
  remote_ip_prefix  = "0.0.0.0/0"
  security_group_id = openstack_networking_secgroup_v2.sg_citrix.id
}

resource "openstack_networking_secgroup_rule_v2" "ica" {
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = 1494
  port_range_max    = 1494
  remote_ip_prefix  = "0.0.0.0/0"
  security_group_id = openstack_networking_secgroup_v2.sg_citrix.id
}

resource "openstack_networking_secgroup_rule_v2" "nrpe" {
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = 5666
  port_range_max    = 5666
  remote_ip_prefix  = var.network_cidr
  security_group_id = openstack_networking_secgroup_v2.sg_citrix.id
}

resource "openstack_networking_secgroup_rule_v2" "https_storefront" {
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = 443
  port_range_max    = 443
  remote_ip_prefix  = "0.0.0.0/0"
  security_group_id = openstack_networking_secgroup_v2.sg_citrix.id
}

resource "openstack_networking_secgroup_rule_v2" "internal_all" {
  direction         = "ingress"
  ethertype         = "IPv4"
  remote_ip_prefix  = var.network_cidr
  security_group_id = openstack_networking_secgroup_v2.sg_citrix.id
}

# SG Centreon
resource "openstack_networking_secgroup_v2" "sg_centreon" {
  name        = "sg-centreon"
  description = "Security group pour Centreon Central"
}

resource "openstack_networking_secgroup_rule_v2" "centreon_ssh" {
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = 22
  port_range_max    = 22
  remote_ip_prefix  = "0.0.0.0/0"
  security_group_id = openstack_networking_secgroup_v2.sg_centreon.id
}

resource "openstack_networking_secgroup_rule_v2" "centreon_web" {
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = 80
  port_range_max    = 80
  remote_ip_prefix  = "0.0.0.0/0"
  security_group_id = openstack_networking_secgroup_v2.sg_centreon.id
}

resource "openstack_networking_secgroup_rule_v2" "centreon_https" {
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = 443
  port_range_max    = 443
  remote_ip_prefix  = "0.0.0.0/0"
  security_group_id = openstack_networking_secgroup_v2.sg_centreon.id
}

resource "openstack_networking_secgroup_rule_v2" "centreon_internal" {
  direction         = "ingress"
  ethertype         = "IPv4"
  remote_ip_prefix  = var.network_cidr
  security_group_id = openstack_networking_secgroup_v2.sg_centreon.id
}
```

### `vms/vm-ddc.tf` — Delivery Controller

```hcl
resource "openstack_compute_instance_v2" "vm_ddc" {
  name            = "vm-ddc"
  image_name      = var.windows_image
  flavor_name     = "a8-ram16-disk100-perf1"
  key_pair        = var.keypair_name
  security_groups = [openstack_networking_secgroup_v2.sg_citrix.name]

  network {
    uuid        = openstack_networking_network_v2.citrix_net.id
    fixed_ip_v4 = "192.168.100.10"
  }

  user_data = <<-EOF
    #ps1_sysnative
    # Configurer le mot de passe admin
    net user Administrator "${var.admin_password}"

    # Renommer le serveur
    Rename-Computer -NewName "DDC01" -Force

    # Activer WinRM
    Enable-PSRemoting -Force
    Set-Item WSMan:\localhost\Client\TrustedHosts -Value "*" -Force

    # Désactiver le pare-feu temporairement (lab)
    Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False

    Restart-Computer -Force
  EOF

  metadata = {
    role = "citrix-delivery-controller"
    env  = "lab"
  }
}
```

### `vms/vm-storefront.tf`

```hcl
resource "openstack_compute_instance_v2" "vm_storefront" {
  name            = "vm-storefront"
  image_name      = var.windows_image
  flavor_name     = "a4-ram8-disk60-perf1"
  key_pair        = var.keypair_name
  security_groups = [openstack_networking_secgroup_v2.sg_citrix.name]

  network {
    uuid        = openstack_networking_network_v2.citrix_net.id
    fixed_ip_v4 = "192.168.100.11"
  }

  user_data = <<-EOF
    #ps1_sysnative
    net user Administrator "${var.admin_password}"
    Rename-Computer -NewName "SF01" -Force
    Enable-PSRemoting -Force
    Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False
    Restart-Computer -Force
  EOF

  metadata = {
    role = "citrix-storefront"
    env  = "lab"
  }
}
```

### `vms/vm-sql.tf`

```hcl
resource "openstack_compute_instance_v2" "vm_sql" {
  name            = "vm-sql"
  image_name      = var.windows_image
  flavor_name     = "a4-ram8-disk80-perf1"
  key_pair        = var.keypair_name
  security_groups = [openstack_networking_secgroup_v2.sg_citrix.name]

  network {
    uuid        = openstack_networking_network_v2.citrix_net.id
    fixed_ip_v4 = "192.168.100.12"
  }

  user_data = <<-EOF
    #ps1_sysnative
    net user Administrator "${var.admin_password}"
    Rename-Computer -NewName "SQL01" -Force
    Enable-PSRemoting -Force
    Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False
    Restart-Computer -Force
  EOF

  metadata = {
    role = "sql-server"
    env  = "lab"
  }
}
```

### `vms/vm-vda.tf`

```hcl
resource "openstack_compute_instance_v2" "vm_vda" {
  name            = "vm-vda"
  image_name      = var.windows_image
  flavor_name     = "a4-ram8-disk60-perf1"
  key_pair        = var.keypair_name
  security_groups = [openstack_networking_secgroup_v2.sg_citrix.name]

  network {
    uuid        = openstack_networking_network_v2.citrix_net.id
    fixed_ip_v4 = "192.168.100.13"
  }

  user_data = <<-EOF
    #ps1_sysnative
    net user Administrator "${var.admin_password}"
    Rename-Computer -NewName "VDA01" -Force
    Enable-PSRemoting -Force
    Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False
    Restart-Computer -Force
  EOF

  metadata = {
    role = "citrix-vda"
    env  = "lab"
  }
}
```

### `vms/vm-centreon.tf`

```hcl
resource "openstack_compute_instance_v2" "vm_centreon" {
  name            = "vm-centreon"
  image_name      = var.linux_image
  flavor_name     = "a2-ram4-disk50-perf1"
  key_pair        = var.keypair_name
  security_groups = [openstack_networking_secgroup_v2.sg_centreon.name]

  network {
    uuid        = openstack_networking_network_v2.citrix_net.id
    fixed_ip_v4 = "192.168.100.20"
  }

  user_data = <<-EOF
    #!/bin/bash
    # Mise à jour système
    dnf update -y

    # Dépôts Centreon
    dnf install -y dnf-plugins-core
    dnf config-manager --add-repo https://packages.centreon.com/rpm-standard/23.10/el9/centreon-23.10.repo
    dnf install -y centreon-full

    # Démarrer les services
    systemctl enable --now mariadb
    systemctl enable --now php-fpm centreon

    # Mot de passe MariaDB root
    mysql -e "ALTER USER 'root'@'localhost' IDENTIFIED BY 'Centreon@Lab2026!'; FLUSH PRIVILEGES;"

    echo "Centreon installé - accès : http://$(hostname -I | awk '{print $1}')/centreon"
  EOF

  metadata = {
    role = "centreon-central"
    env  = "lab"
  }
}
```

### `outputs.tf`

```hcl
output "ip_ddc" {
  description = "IP flottante Delivery Controller"
  value       = openstack_networking_floatingip_v2.fip_ddc.address
}

output "ip_centreon" {
  description = "IP flottante Centreon (interface web)"
  value       = openstack_networking_floatingip_v2.fip_centreon.address
}

output "ip_storefront_private" {
  value = openstack_compute_instance_v2.vm_storefront.network[0].fixed_ip_v4
}

output "ip_sql_private" {
  value = openstack_compute_instance_v2.vm_sql.network[0].fixed_ip_v4
}

output "ip_vda_private" {
  value = openstack_compute_instance_v2.vm_vda.network[0].fixed_ip_v4
}

output "centreon_url" {
  value = "http://${openstack_networking_floatingip_v2.fip_centreon.address}/centreon"
}
```

---

## Déploiement Terraform

```bash
# Initialiser
terraform init

# Valider
terraform validate

# Vérifier le plan (preview)
terraform plan -var-file="terraform.tfvars"

# Appliquer
terraform apply -var-file="terraform.tfvars" -auto-approve

# Récupérer les IPs
terraform output
```

> ⏱️ Le déploiement prend environ **5–8 minutes**. Les VMs Windows ont besoin d'un redémarrage supplémentaire après le cloud-init (~3 min de plus avant d'être joignables en RDP).

---

## Configuration post-déploiement

### Étape 1 — SQL Server sur vm-sql

Se connecter en RDP à vm-sql (via l'IP flottante du DDC en rebond, ou ajouter une fip temporaire) :

```powershell
# Télécharger SQL Server 2022 Express
Invoke-WebRequest -Uri "https://go.microsoft.com/fwlink/?linkid=2215160" `
  -OutFile "C:\SQL2022Express.exe"

# Installer silencieusement
C:\SQL2022Express.exe /q /ACTION=Install /FEATURES=SQL `
  /INSTANCENAME=MSSQLSERVER /SQLSVCACCOUNT="NT AUTHORITY\SYSTEM" `
  /SQLSYSADMINACCOUNTS="BUILTIN\Administrators" /TCPENABLED=1 /NPENABLED=0

# Créer la base Citrix
Invoke-Sqlcmd -Query "CREATE DATABASE CitrixSite; CREATE DATABASE CitrixLog; CREATE DATABASE CitrixMonitoring;"
```

### Étape 2 — Citrix CVAD sur vm-ddc

```powershell
# Télécharger Citrix CVAD 2402 LTSR (compte MyCitrix requis)
# https://www.citrix.com/downloads/citrix-virtual-apps-and-desktops/

# Installer le Delivery Controller
.\XenDesktopServerSetup.exe /components controller /nosql /configure_firewall /noreboot /quiet

# Créer le site Citrix
Add-PSSnapin Citrix*
New-XDSite -DatabaseServer "192.168.100.12" -SiteName "CitrixLab" `
           -DatabaseName "CitrixSite" -LoggingDatabaseName "CitrixLog" `
           -MonitorDatabaseName "CitrixMonitoring"
```

### Étape 3 — StoreFront sur vm-storefront

```powershell
# Installer StoreFront
.\XenDesktopServerSetup.exe /components storefront /configure_firewall /quiet

# Configurer via PowerShell
& "C:\Program Files\Citrix\Receiver StoreFront\Scripts\ImportModules.ps1"
Add-STFDeployment -SiteId 1 -HostBaseUrl "https://192.168.100.11"
$store = Add-STFStoreService -VirtualPath "/Citrix/Store" -AuthenticationServiceVirtualPath "/Citrix/Authentication"
```

### Étape 4 — VDA sur vm-vda

```powershell
# Installer le Virtual Delivery Agent
.\XenDesktopVDASetup.exe /components vda /controllers "192.168.100.10" `
  /enable_hdx_ports /optimize /noreboot /quiet

# Vérifier l'enregistrement
Get-BrokerDesktop -RegistrationState Registered
```

---

## Configuration Centreon

### Accès à l'interface web

```
URL  : http://<IP_FLOTTANTE_CENTREON>/centreon
User : admin
Pass : Centreon (défini à l'installation)
```

### Installation du plugin NSClient++ sur les VMs Windows

Sur **chaque VM Windows** (vm-ddc, vm-storefront, vm-sql, vm-vda) :

```powershell
# Télécharger NSClient++
Invoke-WebRequest -Uri "https://github.com/mickem/nscp/releases/download/0.5.2.35/NSCP-0.5.2.35-x64.msi" `
  -OutFile "C:\NSClient.msi"

# Installer
msiexec /i C:\NSClient.msi /quiet ADDLOCAL=ALL `
  CONF_NRPE=1 CONF_WMI=1 CONF_CHECKS=1 `
  ALLOWED_HOSTS="192.168.100.20" `
  NRPE_PWD="CentreonNRPE2026"
```

Fichier `C:\Program Files\NSClient++\nsclient.ini` à adapter :

```ini
[/settings/NRPE/server]
allowed hosts = 192.168.100.20
port = 5666
use ssl = false
verify mode = none

[/modules]
NRPEServer = enabled
CheckSystem = enabled
CheckDisk = enabled
CheckEventLog = enabled
CheckExternalScripts = enabled
```

### Ajouter les hôtes dans Centreon

Dans **Configuration > Hôtes > Ajouter** :

| Nom | Adresse | Template |
|---|---|---|
| vm-ddc | 192.168.100.10 | OS-Windows-NSClient-05-custom |
| vm-storefront | 192.168.100.11 | OS-Windows-NSClient-05-custom |
| vm-sql | 192.168.100.12 | OS-Windows-NSClient-05-custom |
| vm-vda | 192.168.100.13 | OS-Windows-NSClient-05-custom |

### Checks services standards (via NSClient++)

| Check | Commande Centreon |
|---|---|
| CPU | check_nrpe -H $HOSTADDRESS$ -c check_cpu |
| Mémoire RAM | check_nrpe -H $HOSTADDRESS$ -c check_memory |
| Disque C: | check_nrpe -H $HOSTADDRESS$ -c check_drivesize |
| Uptime | check_nrpe -H $HOSTADDRESS$ -c check_uptime |

---

## Plugin Centreon personnalisé — Checks Citrix

### Script PowerShell côté vm-ddc

Créer `C:\Scripts\check_citrix.ps1` :

```powershell
param(
    [Parameter(Mandatory=$true)]
    [ValidateSet("sessions","machines","deliverygroups","registrations")]
    [string]$Check
)

Add-PSSnapin Citrix* -ErrorAction SilentlyContinue

switch ($Check) {

    "sessions" {
        $sessions = (Get-BrokerSession | Measure-Object).Count
        $warn = 20; $crit = 40
        if ($sessions -ge $crit) {
            Write-Host "CRITICAL - $sessions sessions actives | sessions=$sessions;$warn;$crit;0;"
            exit 2
        } elseif ($sessions -ge $warn) {
            Write-Host "WARNING - $sessions sessions actives | sessions=$sessions;$warn;$crit;0;"
            exit 1
        } else {
            Write-Host "OK - $sessions sessions actives | sessions=$sessions;$warn;$crit;0;"
            exit 0
        }
    }

    "machines" {
        $total       = (Get-BrokerMachine | Measure-Object).Count
        $registered  = (Get-BrokerMachine -RegistrationState Registered | Measure-Object).Count
        $unregistered = $total - $registered
        if ($unregistered -ge 2) {
            Write-Host "CRITICAL - $unregistered/$total machines non enregistrées | registered=$registered;1;2;0;$total"
            exit 2
        } elseif ($unregistered -eq 1) {
            Write-Host "WARNING - $unregistered/$total machines non enregistrées | registered=$registered;1;2;0;$total"
            exit 1
        } else {
            Write-Host "OK - $registered/$total machines enregistrées | registered=$registered;1;2;0;$total"
            exit 0
        }
    }

    "deliverygroups" {
        $dgs = Get-BrokerDesktopGroup
        $enabled  = ($dgs | Where-Object {$_.Enabled -eq $true} | Measure-Object).Count
        $disabled = ($dgs | Where-Object {$_.Enabled -eq $false} | Measure-Object).Count
        if ($disabled -gt 0) {
            Write-Host "WARNING - $disabled Delivery Group(s) désactivé(s) | enabled=$enabled;; disabled=$disabled;1;2"
            exit 1
        } else {
            Write-Host "OK - Tous les Delivery Groups actifs ($enabled) | enabled=$enabled"
            exit 0
        }
    }

    "registrations" {
        $failed = (Get-BrokerMachine -RegistrationState Unregistered | Measure-Object).Count
        if ($failed -ge 2) {
            Write-Host "CRITICAL - $failed machines Unregistered | failed=$failed;1;2;;"
            exit 2
        } elseif ($failed -eq 1) {
            Write-Host "WARNING - $failed machine Unregistered | failed=$failed;1;2;;"
            exit 1
        } else {
            Write-Host "OK - Toutes les machines enregistrées | failed=0;1;2;;"
            exit 0
        }
    }
}
```

### Rendre le script accessible via NSClient++

Dans `nsclient.ini` sur vm-ddc, ajouter :

```ini
[/settings/external scripts/scripts]
check_citrix_sessions      = cmd /c echo scripts\check_citrix.ps1 -Check sessions | powershell.exe -command -
check_citrix_machines      = cmd /c echo scripts\check_citrix.ps1 -Check machines | powershell.exe -command -
check_citrix_deliverygroups = cmd /c echo scripts\check_citrix.ps1 -Check deliverygroups | powershell.exe -command -
```

Ou avec la directive propre NSClient++ :

```ini
[/settings/external scripts/scripts]
check_citrix_sessions = powershell -NoProfile -ExecutionPolicy Bypass -File "C:\Scripts\check_citrix.ps1" -Check sessions
check_citrix_machines = powershell -NoProfile -ExecutionPolicy Bypass -File "C:\Scripts\check_citrix.ps1" -Check machines
check_citrix_deliverygroups = powershell -NoProfile -ExecutionPolicy Bypass -File "C:\Scripts\check_citrix.ps1" -Check deliverygroups
```

### Commandes dans Centreon

Dans **Configuration > Commandes > Checks > Ajouter** :

| Nom de la commande | Ligne de commande |
|---|---|
| check_citrix_sessions | `$USER1$/check_nrpe -H $HOSTADDRESS$ -c check_citrix_sessions` |
| check_citrix_machines | `$USER1$/check_nrpe -H $HOSTADDRESS$ -c check_citrix_machines` |
| check_citrix_dg | `$USER1$/check_nrpe -H $HOSTADDRESS$ -c check_citrix_deliverygroups` |

### Services à associer à vm-ddc dans Centreon

| Nom du service | Commande | Warning | Critical |
|---|---|---|---|
| Citrix-Sessions-Actives | check_citrix_sessions | 20 | 40 |
| Citrix-Machines-Enregistrees | check_citrix_machines | — | — |
| Citrix-DeliveryGroups-Status | check_citrix_dg | — | — |
| Citrix-Broker-Service | check_nrpe ... check_service name=CitrixBrokerService | — | — |
| StoreFront-HTTPS | check_tcp -H 192.168.100.11 -p 443 | — | — |
| SQL-Port-1433 | check_tcp -H 192.168.100.12 -p 1433 | — | — |

---

## Dashboard Centreon (vue NOC)

Dans **Monitoring > Dashboard > Créer un dashboard** : "Citrix Lab Supervision"

Widgets recommandés :
- **Status Grid** : état global de tous les hôtes Citrix
- **Graph** : courbe des sessions actives sur 24h (métrique `sessions`)
- **Gauge** : CPU du vm-ddc (métrique `cpu`)
- **Top counter** : services en état CRITICAL/WARNING

---

## Vérification finale

```bash
# Depuis vm-centreon, tester les checks NRPE
/usr/lib/nagios/plugins/check_nrpe -H 192.168.100.10 -c check_citrix_sessions
/usr/lib/nagios/plugins/check_nrpe -H 192.168.100.10 -c check_citrix_machines
/usr/lib/nagios/plugins/check_nrpe -H 192.168.100.10 -c check_cpu

# Vérifier l'état des services dans Centreon
# Monitoring > Services > All services
```

---

## Destruction de l'infrastructure

> ⚠️ Cette commande **supprime toutes les VMs, le réseau et les IPs flottantes** du lab. Bien sauvegarder les scripts et la doc avant.

```bash
terraform destroy -var-file="terraform.tfvars" -auto-approve
```

Vérification dans Infomaniak :

```bash
openstack server list          # doit retourner une liste vide
openstack network list         # citrix-lab-network supprimé
openstack floating ip list     # IPs libérées
```

---

## Résumé des commandes clés

| Étape | Commande |
|---|---|
| Init Terraform | `terraform init` |
| Preview | `terraform plan -var-file="terraform.tfvars"` |
| Déployer | `terraform apply -var-file="terraform.tfvars" -auto-approve` |
| Voir les IPs | `terraform output` |
| Détruire | `terraform destroy -var-file="terraform.tfvars" -auto-approve` |
| Test check Citrix | `/usr/lib/nagios/plugins/check_nrpe -H 192.168.100.10 -c check_citrix_sessions` |

---

## Ressources

- [Citrix CVAD 2402 LTSR — Documentation officielle](https://docs.citrix.com/en-us/citrix-virtual-apps-desktops)
- [Centreon 23.10 — Installation RPM AlmaLinux](https://docs.centreon.com/docs/installation/introduction/)
- [NSClient++ documentation](https://docs.nsclient.org/)
- [Terraform OpenStack Provider](https://registry.terraform.io/providers/terraform-provider-openstack/openstack/latest/docs)
- [Infomaniak Public Cloud — API OpenStack](https://api.pub1.infomaniak.cloud)
- [Cloudbase-Init Windows images QCOW2](https://cloudbase.it/windows-cloud-images/)

---

*Documentation rédigée par Alexandra ASSAGA - Lab Citrix+Centreon sur Infomaniak OpenStack*