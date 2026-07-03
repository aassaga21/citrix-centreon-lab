# TP : Supervision d'une infrastructure Citrix VDI avec Centreon sur Infomaniak OpenStack

> **Auteur :** Alexandra ASSAGA  
> **GitHub :** [aassaga21](https://github.com/aassaga21)  
> **Date :** Juillet 2026  
> **Durée estimée :** 2–3 jours  
> **Tags :** `Terraform` `Citrix CVAD` `Centreon` `Infomaniak` `OpenStack` `Supervision` `VDI`

---

## Introduction

Ce TP porte sur la mise en place d'une supervision applicative et infrastructure pour un environnement Citrix VDI (Delivery Controller, StoreFront, VDA) hébergé sur Infomaniak OpenStack. L'objectif est de déployer une chaîne de collecte Centreon (serveur central + poller) capable de remonter en temps réel l'état des composants Citrix via SNMP, WMI et NRPE, afin de garantir la disponibilité du service de bureau virtuel pour les utilisateurs finaux.

---

## Objectifs pédagogiques

- Déployer une infrastructure VDI Citrix CVAD via **Terraform** sur Infomaniak Public Cloud (OpenStack)
- Superviser les composants Citrix avec **Centreon** (checks système + checks métier via PowerShell)
- Maîtriser le cycle complet **provision → configurer → superviser → détruire** (`terraform destroy`)
- Créer des plugins de supervision Nagios/Centreon personnalisés pour Citrix

---

## Architecture cible

![image](https://hackmd.io/_uploads/SkXWYLfXze.png)

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
terraform --version   # <= 1.7
openstack --version   # CLI OpenStack
python3 --version
function python3 { python @args } # Option simple : dans PowerShell, ajoute un alias dans ton profil (notepad $PROFILE) :
```
![image](https://hackmd.io/_uploads/HyvW9LfmMx.png)
![image](https://hackmd.io/_uploads/BkmK5Iz7fg.png)

### Récupérer les credentials OpenStack

```bash
# Depuis le dashboard Infomaniak > API Access > Télécharger openrc.sh

# Fichier RC Infomaniak (valeurs de référence)

export OS_AUTH_URL=https://api.pub1.infomaniak.cloud/identity/v3
export OS_PROJECT_NAME=TON_PROJECT_NAME_INFOMANIAK
export OS_PROJECT_DOMAIN_NAME=default
export OS_USERNAME=TON_USERNAME_INFOMANIAK
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_ID=TON_PROJECT_ID
export OS_IDENTITY_API_VERSION=3
export OS_INTERFACE=public
export OS_REGION_NAME=dc3-a
```

> Télécharger le fichier RC depuis : Infomaniak Public Cloud → ton nom → **"Télécharger le fichier RC OpenStack"**

#### Chiffrement des credentials Infomaniak

### Contexte
Les credentials Infomaniak ne doivent jamais être stockés en clair dans un fichier.
Le mot de passe est chiffré via PowerShell et stocké dans `secret.key`, lisible uniquement par ta session Windows.

### Étape 1: Autoriser l'exécution des scripts PowerShell

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```
![image](https://hackmd.io/_uploads/S1DgaFtZMg.png)

### Étape 2: Chiffrer le mot de passe (une seule fois)

```powershell
Read-Host "Entre ton mot de passe" -AsSecureString | ConvertFrom-SecureString | Out-File "C:\Users\alexa\secret.key"
```
![image](https://hackmd.io/_uploads/Hy5WCFYbzl.png)

→ Tape ton mot de passe Infomaniak → Entrée → `secret.key` est créé chiffré

### Étape 3: Fichier `infomaniak-credentials.ps1`

```powershell
# ===== INFOMANIAK OPENSTACK =====
$env:OS_AUTH_URL              = "https://api.pub1.infomaniak.cloud/identity/v3"
$env:OS_PROJECT_NAME          = "TON_PROJECT_NAME_INFOMANIAK"
$env:OS_PROJECT_ID            = "TON_PROJECT_ID"
$env:OS_PROJECT_DOMAIN_NAME   = "default"
$env:OS_USERNAME              = "TON_USERNAME_INFOMANIAK"
$env:OS_USER_DOMAIN_NAME      = "default"
$securePassword               = Get-Content "C:\Users\alexa\secret.key" | ConvertTo-SecureString
$credential                   = New-Object System.Management.Automation.PSCredential("user", $securePassword)
$env:OS_PASSWORD              = $credential.GetNetworkCredential().Password
$env:OS_REGION_NAME           = "dc3-a"
$env:OS_INTERFACE             = "public"
$env:OS_IDENTITY_API_VERSION  = "3"

Write-Host "Variables Infomaniak chargees !" -ForegroundColor Green
Write-Host "Projet : $env:OS_PROJECT_NAME" -ForegroundColor White
Write-Host "User   : $env:OS_USERNAME" -ForegroundColor White
```
![image](https://hackmd.io/_uploads/HyZXVWsGMl.png)

### Étape 4: Charger les variables (à chaque session)

```powershell
. .\infomaniak-credentials.ps1
```

![image](https://hackmd.io/_uploads/HkXK4-ozfg.png)

### Étape 5: Vérifier la connexion

```powershell
openstack server list
openstack image list
```
![image](https://hackmd.io/_uploads/BJu4AtK-Mx.png)
![image](https://hackmd.io/_uploads/SJZl1lX7fx.png)
![image](https://hackmd.io/_uploads/SJCCCy7mMe.png)

### Étape 6: création de la clé SSH

```
# Génère une nouvelle paire de clés localement avec ssh-keygen
cd C:\Users\alexa\citrix-centreon-lab
ssh-keygen -t ed25519 -f alex-lab-key -C "alex-lab-key" -N '""'

# Importe la clé publique dans OpenStack
openstack keypair create alex-lab-key --public-key alex-lab-key.pub

# Restreins les permissions de la clé privée
icacls alex-lab-key /inheritance:r
icacls alex-lab-key /grant:r "$($env:USERNAME):(R)"

# Vérifie la correspondance avant d'aller plus loin
ssh-keygen -E md5 -lf alex-lab-key
openstack keypair show alex-lab-key
```

![image](https://hackmd.io/_uploads/BJCcu147Me.png)
![image](https://hackmd.io/_uploads/rknF5yNXfl.png)
![image](https://hackmd.io/_uploads/BJk7jJNXzl.png)
![image](https://hackmd.io/_uploads/r1dKh1Vmzx.png)
![image](https://hackmd.io/_uploads/r1vv31Vmfg.png)

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
├── vm-ddc.tf
├── vm-storefront.tf
├── vm-sql.tf
├── vm-vda.tf
└── vm-centreon.tf
└── scripts/
    ├── cloud-init-windows.ps1
    └── cloud-init-centreon.sh
```

```
mkdir citrix-centreon-lab
cd citrix-centreon-lab
nano main.tf
```
![image](https://hackmd.io/_uploads/B1QMEg7QGe.png)
![image](https://hackmd.io/_uploads/HJi63omQzg.png)

---

## Terraform : Code complet

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
  default     = "Windows Server 2022 Standard"
}

variable "linux_image" {
  description = "Image Rocky Linux 9 pour Centreon"
  type        = string
  default     = "Rocky Linux 9"
}

variable "admin_password" {
  description = "Mot de passe administrateur Windows"
  type        = string
  sensitive   = true
  default     = "TON_MOT_DE_PASSE"
}

variable "network_cidr" {
  type    = string
  default = "192.168.100.0/24"
}
```

![image](https://hackmd.io/_uploads/SkzNZjmQze.png)


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
      version = "~> 1.53.0"
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

![image](https://hackmd.io/_uploads/ByqC-NmmMx.png)

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
    start = "192.168.100.50"
    end   = "192.168.100.100"
  }
}

# Router vers l'extérieur
data "openstack_networking_network_v2" "ext_net" {
  name = "ext-floating1"
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
  pool = "ext-floating1"
}

resource "openstack_networking_floatingip_v2" "fip_centreon" {
  pool = "ext-floating1"
}

# Association des floating IPs aux instances
resource "openstack_compute_floatingip_associate_v2" "fip_ddc_assoc" {
  floating_ip = openstack_networking_floatingip_v2.fip_ddc.address
  instance_id = openstack_compute_instance_v2.vm_ddc.id
}

resource "openstack_compute_floatingip_associate_v2" "fip_centreon_assoc" {
  floating_ip = openstack_networking_floatingip_v2.fip_centreon.address
  instance_id = openstack_compute_instance_v2.vm_centreon.id
}
```

![image](https://hackmd.io/_uploads/SkMam3XQfg.png)

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

![image](https://hackmd.io/_uploads/BJZTI4X7fg.png)

### `vm-ddc.tf` — Delivery Controller

```hcl
resource "openstack_compute_instance_v2" "vm_ddc" {
  name            = "vm-ddc"
  image_name      = var.windows_image
  flavor_name     = "a8-ram16-disk80-perf1"
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

![image](https://hackmd.io/_uploads/S1lWRoX7Gl.png)

### `vm-storefront.tf`

```hcl
resource "openstack_compute_instance_v2" "vm_storefront" {
  name            = "vm-storefront"
  image_name      = var.windows_image
  flavor_name     = "a4-ram8-disk80-perf1"
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

![image](https://hackmd.io/_uploads/rkSuy3mQGg.png)

### `vm-sql.tf`

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

![image](https://hackmd.io/_uploads/r10xYE7Xzx.png)

### `vm-vda.tf`

```hcl
resource "openstack_compute_instance_v2" "vm_vda" {
  name            = "vm-vda"
  image_name      = var.windows_image
  flavor_name     = "a4-ram8-disk80-perf1"
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

![image](https://hackmd.io/_uploads/HkBJx377Me.png)

### `vm-centreon.tf`

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
    dnf update -y
    dnf install -y dnf-plugins-core
    dnf config-manager --add-repo https://packages.centreon.com/rpm-standard/25.10/el9/centreon-25.10.repo
    dnf clean all --enablerepo=*
    dnf install -y centreon-mariadb centreon
    systemctl daemon-reload
    systemctl restart mariadb
    systemctl enable --now mariadb php-fpm httpd centreon centreon-mariadb centengine cbd centreontrapd gorgoned
    mysql -e "ALTER USER 'root'@'localhost' IDENTIFIED BY 'Centreon@Lab2026!'; FLUSH PRIVILEGES;"
    firewall-cmd --permanent --add-service=http
    firewall-cmd --permanent --add-service=https
    firewall-cmd --reload
    echo "Centreon installe - acces : http://$(hostname -I | awk '{print $1}')/centreon"
  EOF
  
  metadata = {
    role = "centreon-central"
    env  = "lab"
  }
}
```

![image](https://hackmd.io/_uploads/B1hBoNmXzg.png)

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

![image](https://hackmd.io/_uploads/rkfGvN77Ml.png)

---

## Déploiement Terraform

```bash
# Initialiser
terraform init

# Déplace tous les fichiers de vms/ à la racine du projet.
Move-Item vms\*.tf .
Remove-Item vms -Recurse

# Valider
terraform validate

# Vérifier le plan (preview)
terraform plan -var-file="terraform.tfvars"

# Appliquer
terraform apply -var-file="terraform.tfvars" -auto-approve

# Récupérer les IPs
terraform output
```

![image](https://hackmd.io/_uploads/S1o5i4m7Ge.png)
![image](https://hackmd.io/_uploads/HySfJr7XMl.png)
![image](https://hackmd.io/_uploads/r11U1rQmGx.png)
![image](https://hackmd.io/_uploads/rkKYJH77Mx.png)
![image](https://hackmd.io/_uploads/ryQs1B7XMx.png)
![image](https://hackmd.io/_uploads/B1OCkSQ7zl.png)
![image](https://hackmd.io/_uploads/rkxGf2QQfg.png)
![image](https://hackmd.io/_uploads/BymqB2XXzx.png)
![image](https://hackmd.io/_uploads/BJUgL3mmze.png)
![image](https://hackmd.io/_uploads/ByYYUh7XMx.png)

> Le déploiement prend environ **5–8 minutes**. Les VMs Windows ont besoin d'un redémarrage supplémentaire après le cloud-init (~3 min de plus avant d'être joignables en RDP).

```
# Recrée toutes les instances avec cette clé enfin stable
terraform apply -replace="openstack_compute_instance_v2.vm_ddc" -replace="openstack_compute_instance_v2.vm_storefront" -replace="openstack_compute_instance_v2.vm_sql" -replace="openstack_compute_instance_v2.vm_vda" -replace="openstack_compute_instance_v2.vm_centreon" -var-file="terraform.tfvars" -auto-approve

# Une fois terminé (attends 3-5 min), reteste
terraform output ip_centreon
ssh-keygen -R <IP_AFFICHEE>
ssh -i alex-lab-key rocky@<IP_AFFICHEE>

```
![image](https://hackmd.io/_uploads/SyT261EmGl.png)
![image](https://hackmd.io/_uploads/H1SKRyEmMg.png)
![image](https://hackmd.io/_uploads/By2LyeEXzl.png)

> Tu es connecté sur `vm-centreon`. Maintenant on diagnostique pourquoi le web ne répondait pas.

**1. Vérifie que le cloud-init s'est bien terminé sans erreur**
```bash
sudo cloud-init status
```

![image](https://hackmd.io/_uploads/ryQVfxVmfg.png)

Si ça affiche `status: running`, patiente encore quelques minutes. Si `status: error`, regarde le détail :

```bash
sudo cat /var/log/cloud-init-output.log | tail -100
```
![image](https://hackmd.io/_uploads/B1ZnflEmMx.png)
![image](https://hackmd.io/_uploads/ryZZXeNXGx.png)

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

### Installation de centreon

> Le script d'installation automatique prévu dans le `user_data` Terraform (dépôt `centreon-23.10`, paquet `centreon-full`) s'est avéré obsolète. L'installation a donc été réalisée manuellement en SSH sur `vm-centreon`, ce qui a permis d'identifier et de corriger plusieurs incompatibilités liées à la version actuelle de Centreon (25.10) sur Rocky Linux 9.

### Connexion à la VM

```bash
ssh -i alex-lab-key rocky@195.15.246.234
```
![image](https://hackmd.io/_uploads/r14epOVmzx.png)

### Étape 1 — Mise à jour du système et prérequis

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf install -y epel-release
sudo dnf config-manager --set-enabled crb
sudo dnf clean all --enablerepo=*
```
![image](https://hackmd.io/_uploads/S1tQpOVXGe.png)
![image](https://hackmd.io/_uploads/SkYSaOEQGg.png)
![image](https://hackmd.io/_uploads/S1iU6OV7fg.png)
![image](https://hackmd.io/_uploads/BJzpp_VXGl.png)

> `epel-release` fournit les dépendances Perl requises par Centreon Gorgone (`JSON::XS`, `DateTime`, `YAML::XS`...). Le dépôt **CRB** (CodeReady Builder) doit être activé en complément, sinon certaines dépendances de compilation d'EPEL restent indisponibles.

### Étape 2 — Ajout du dépôt Centreon (version 25.10)

```bash
sudo dnf config-manager --add-repo https://packages.centreon.com/rpm-standard/25.10/el9/centreon-25.10.repo
sudo dnf clean all --enablerepo=*
```

![image](https://hackmd.io/_uploads/SymWC_Nmfg.png)
![image](https://hackmd.io/_uploads/H1Bz0u4QMl.png)

> Le dépôt `23.10` documenté initialement n'est plus disponible (erreur 404). La version stable actuelle est la **25.10**.

### Étape 3 — Activation du module PHP 8.2

```bash
sudo dnf module reset php -y
sudo dnf module enable php:8.2 -y
```
![image](https://hackmd.io/_uploads/Hk-4RuE7Ge.png)

> Sur EL9, PHP est géré par les modules DNF : aucun flux n'est activé par défaut, ce qui bloque l'installation de `php-common` (requis par `centreon-web`) tant qu'un module n'est pas explicitement activé.

### Étape 4 — Installation de Centreon

```bash
sudo dnf install -y centreon-mariadb centreon
```

![image](https://hackmd.io/_uploads/SyP1RdVmGg.png)
![image](https://hackmd.io/_uploads/ryQr0uNXMg.png)

### Étape 5 — Activation des services

```bash
sudo systemctl daemon-reload
sudo systemctl restart mariadb
sudo systemctl enable --now mariadb php-fpm httpd centreon centengine cbd centreontrapd gorgoned
```

![image](https://hackmd.io/_uploads/HylL0uEmzx.png)

> `centreon-mariadb` est un paquet de configuration (pas un service systemd) : il ne doit pas être inclus dans la commande `enable --now`.

### Étape 6 — Résolution du blocage PHP-FPM (SELinux)

Au premier démarrage, `php-fpm` échouait avec l'erreur `unable to bind` (code 13 — permission refusée), causée par SELinux en mode `enforcing` :

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
sudo systemctl restart php-fpm
```
![image](https://hackmd.io/_uploads/BJMsRONQfx.png)
![image](https://hackmd.io/_uploads/HyBcCuN7Ge.png)
![image](https://hackmd.io/_uploads/B14aRdEQfe.png)

> Centreon recommande officiellement SELinux en mode **permissive** pour ce type d'installation.

### Étape 7 — Configuration du mot de passe root MariaDB et vérification

```bash
sudo mysql -e "ALTER USER 'root'@'localhost' IDENTIFIED BY 'Ton_mot_de_passe'; FLUSH PRIVILEGES;"
sudo mysql -u root -p centreon
```
![image](https://hackmd.io/_uploads/Hkc1gKNXGe.png)

### Étape 8 — Vérification réseau

```bash
sudo ss -tlnp | grep :80
```
![image](https://hackmd.io/_uploads/rJmlJFV7fe.png)

> Aucun `firewalld` n'est présent sur cette image Rocky Linux, les security groups OpenStack (`sg-centreon`, ports 80/443/22 ouverts en `0.0.0.0/0`) suffisent pour l'accès externe, confirmé par `openstack security group rule list sg-centreon` et `Test-NetConnection -Port 80`.

### Étape 9 — Assistant d'installation web

Accès à l'assistant :

![image](https://hackmd.io/_uploads/HyXZkKE7fx.png)
![image](https://hackmd.io/_uploads/Sy5bJKE7Ml.png)
![image](https://hackmd.io/_uploads/SkEGJKVXfl.png)
![image](https://hackmd.io/_uploads/H1mEJFE7Gx.png)
![image](https://hackmd.io/_uploads/SJMSyt4Qzx.png)
![image](https://hackmd.io/_uploads/H1hSktVXzl.png)
![image](https://hackmd.io/_uploads/SyE8kYNmfx.png)
![image](https://hackmd.io/_uploads/H1kD1YEXMe.png)
![image](https://hackmd.io/_uploads/SyIv1FV7Me.png)
![image](https://hackmd.io/_uploads/HyyOyK47fl.png)

### Accès à l'interface web

```
URL  : http://<IP_FLOTTANTE_CENTREON>/centreon
User : admin
Pass : Ton_mot_de_passe (défini à l'étape 5 de l'assistant d'installation )
```
![image](https://hackmd.io/_uploads/r1hd1KE7ze.png)
![image](https://hackmd.io/_uploads/ryepkF4Qzx.png)

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

![image](https://hackmd.io/_uploads/rJNnMYV7Ml.png)
![image](https://hackmd.io/_uploads/r1u6ztN7Gx.png)

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

> Cette commande **supprime toutes les VMs, le réseau et les IPs flottantes** du lab. Bien sauvegarder les scripts et la doc avant.

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

## Conclusion
Ce TP a permis de construire une solution de supervision complète couvrant les trois couches critiques d'un environnement VDI : l'accès (StoreFront), l'orchestration des sessions (Delivery Controller) et l'exécution (VDA). L'intégration avec Centreon, hébergée sur la même infrastructure Infomaniak OpenStack, offre une visibilité centralisée via dashboards et alertes, réduisant le délai de détection d'incident. Cette architecture est extensible : ajout de sondes applicatives supplémentaires, seuils personnalisés par machine virtuelle, ou intégration future avec un système de notification (email, Slack) pour renforcer la réactivité opérationnelle.

---

## Ressources

- [Citrix CVAD 2402 LTSR — Documentation officielle](https://docs.citrix.com/en-us/citrix-virtual-apps-desktops)
- [Centreon 23.10 — Installation RPM AlmaLinux](https://docs.centreon.com/docs/installation/introduction/)
- [NSClient++ documentation](https://docs.nsclient.org/)
- [Terraform OpenStack Provider](https://registry.terraform.io/providers/terraform-provider-openstack/openstack/latest/docs)
- [Infomaniak Public Cloud — API OpenStack](https://api.pub1.infomaniak.cloud)
- [Infomaniak Public Cloud — Catalogue d'images](https://www.infomaniak.com/fr/hebergement/public-cloud)
- [Terraform Provider Openstack](https://registry.terraform.io/providers/terraform-provider-openstack/openstack/latest/docs)

---