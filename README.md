# Rapport de TP : Virtualisation imbriquée avec un hyperviseur de type 2

**Chapitre 1 : Manipuler un hyperviseur de type 2**

Réalisé par : Romaissa Meliani | Octobre 2026

## Sommaire

1. [Introduction et objectif](#1-introduction-et-objectif)
2. [Architecture réalisée](#2-architecture-réalisée)
3. [Caractéristiques de la machine physique](#3-caractéristiques-de-la-machine-physique)
4. [Outils et technologies utilisés](#4-outils-et-technologies-utilisés)
5. [Configuration des machines virtuelles](#5-configuration-des-machines-virtuelles)
6. [Déroulement de la réalisation](#6-déroulement-de-la-réalisation)
7. [Tests et vérifications](#7-tests-et-vérifications)
8. [Problèmes rencontrés et solutions](#8-problèmes-rencontrés-et-solutions)
9. [Limites et précautions](#9-limites-et-précautions)
10. [Conclusion](#10-conclusion)

## 1. Introduction et objectif

Ce TP a pour but de monter une architecture de **virtualisation imbriquée** (nested virtualization) sur une seule machine physique. Un hyperviseur de type 2 installé sur Windows héberge deux machines virtuelles de niveau 1. Chacune de ces machines joue à son tour le rôle d'hyperviseur et héberge deux machines virtuelles de niveau 2, soit six machines virtuelles au total.

L'énoncé demandait l'hyperviseur Oracle VirtualBox 6.1 et prévoyait VMware Workstation comme solution de secours en cas d'échec. J'ai choisi **VMware Workstation Pro** dès le départ, car il gère mieux la virtualisation imbriquée sous Windows 11.

Toutes les machines devaient être en mode **bridge**, afin d'obtenir une adresse IP sur le même réseau que mon téléphone (point d'accès Wi-Fi) et de pouvoir être consultées depuis un autre appareil, par exemple le téléphone de l'enseignant.

## 2. Architecture réalisée

Le numéro de chaque machine de niveau 2 indique la machine de niveau 1 qui l'héberge.

| Niveau | Machine | Système | Rôle |
|---|---|---|---|
| Physique | PC portable Dell Latitude 5300 | Windows 11 Pro | Héberge VMware Workstation |
| Niveau 1 | **VM1** | Proxmox VE 7.4 | Hyperviseur de niveau 1 |
| Niveau 1 | **VM2** | Ubuntu 26.10 + KVM | Hyperviseur de niveau 1 |
| Niveau 2 (dans VM1) | **VM11** | Ubuntu Server 26.10 | Invité de VM1 |
| Niveau 2 (dans VM1) | **VM12** | Ubuntu Server 26.10 | Invité de VM1 |
| Niveau 2 (dans VM2) | **VM21** | Proxmox VE 7.4 | Invité de VM2 |
| Niveau 2 (dans VM2) | **VM22** | Proxmox VE 7.4 | Invité de VM2 |

```text
PC physique (Windows 11, VMware Workstation Pro)
├── VM1 : Proxmox VE 7.4
│   ├── VM11 : Ubuntu Server 26.10
│   └── VM12 : Ubuntu Server 26.10
└── VM2 : Ubuntu 26.10 + KVM/libvirt
    ├── VM21 : Proxmox VE 7.4
    └── VM22 : Proxmox VE 7.4
```

Au départ, j'avais inversé les deux côtés du schéma en installant un Proxmox à l'intérieur de VM1. Après relecture de l'énoncé, j'ai supprimé cette machine et reconstruit l'architecture dans le bon sens.

## 3. Caractéristiques de la machine physique

| Élément | Valeur |
|---|---|
| Modèle | Dell Latitude 5300 |
| Processeur | Intel Core i5-8265U, 4 coeurs, 8 threads logiques, 1,60 GHz de base |
| Mémoire vive | 16 Go (15,8 Go utilisables), 2400 MT/s |
| Carte graphique | Intel UHD Graphics 620 |
| Stockage | SSD NVMe de 238 Go |
| Système d'exploitation | Windows 11 Pro, version 25H2 |
| Virtualisation matérielle | Intel VT-x activée dans le BIOS (Virtualization : Enabled) |
| Carte Wi-Fi | Intel Wireless-AC 9560 160 MHz |
| Réseau utilisé | Point d'accès Wi-Fi du téléphone, réseau 10.84.119.0/24 |

## 4. Outils et technologies utilisés

**Hyperviseurs et virtualisation**

- **VMware Workstation Pro** (hyperviseur de type 2), avec l'option *Virtualize Intel VT-x/EPT* pour la virtualisation imbriquée.
- **Proxmox VE 7.4** (basé sur Debian 11, noyau 5.15), qui s'appuie sur KVM/QEMU et propose une interface web sur le port 8006.
- **KVM/QEMU et libvirt** dans Ubuntu, pilotés par **virt-manager** (Gestionnaire de machines virtuelles).

**Systèmes d'exploitation**

- Windows 11 Pro (machine physique).
- Ubuntu Server 26.10 (VM2, VM11, VM12), avec le bureau léger **Xubuntu** (XFCE) ajouté sur VM2 pour utiliser virt-manager.
- Proxmox VE 7.4 (VM1, VM21, VM22).

**Réseau et services**

- Mode **Bridged** de VMware (VMnet0 lié à la carte Wi-Fi), pont Linux **vmbr0** dans Proxmox et pont **br0** dans Ubuntu.
- **Netplan** et **systemd-networkd** pour configurer le pont dans Ubuntu.
- **Apache2** installé sur les machines Ubuntu pour afficher une page web identifiant chaque machine.
- **SSH et scp** pour copier l'image ISO de Proxmox de Windows vers VM2.
- **Cockpit** et son module **cockpit-machines** : interface web de gestion des serveurs Linux (port 9090), utilisée sur les machines Ubuntu pour les superviser depuis un navigateur. Sur VM2, elle permet aussi de gérer les machines vm21 et vm22 (KVM/libvirt).

**Outils de contrôle**

- Windows PowerShell (`ipconfig`, `ping`, `scp`), console web noVNC de Proxmox et console SPICE de virt-manager.
- Commandes `kvm-ok`, `egrep -c '(vmx|svm)' /proc/cpuinfo`, `ip a`, `ip route`, `ping` et `virsh`.

## 5. Configuration des machines virtuelles

### 5.1 Répartition des ressources

La mémoire de la machine physique est limitée (16 Go) et Windows en consomme environ 5 Go. J'ai donc réparti les ressources avec prudence. Les disques sont en **allocation dynamique** (thin provisioning), car il restait seulement une cinquantaine de Go libres.

| Machine | vCPU | RAM | Disque | Hébergée par |
|---|---|---|---|---|
| VM1 (Proxmox) | 2 | 4 Go | 25 Go | VMware Workstation |
| VM2 (Ubuntu) | 2 | 6 Go | 40 Go | VMware Workstation |
| VM11 (Ubuntu) | 1 | 1 Go | 10 Go | VM1 |
| VM12 (Ubuntu) | 1 | 1 Go | 10 Go | VM1 |
| VM21 (Proxmox) | 1 | 2 Go | 20 Go | VM2 (KVM) |
| VM22 (Proxmox) | 1 | 2 Go | 20 Go | VM2 (KVM) |

Les machines de niveau 2 utilisent le processeur en mode **host-passthrough** (VM21 et VM22), afin que les instructions de virtualisation soient transmises jusqu'à elles.

### 5.2 Plan d'adressage IP

Tout le réseau est en pont sur le point d'accès : réseau **10.84.119.0/24**, passerelle **10.84.119.68**. Mon PC Windows a l'adresse 10.84.119.193.

| Machine | Adresse IP | Attribution | Accès |
|---|---|---|---|
| VM1 (Proxmox) | 10.84.119.201 | Fixe | https://10.84.119.201:8006 |
| VM11 (Ubuntu) | 10.84.119.242 | DHCP | http://10.84.119.242 |
| VM12 (Ubuntu) | 10.84.119.63 | DHCP | http://10.84.119.63 |
| VM2 (Ubuntu) | 10.84.119.81 | DHCP | http://10.84.119.81 |
| VM21 (Proxmox) | 10.84.119.202 | Fixe | https://10.84.119.202:8006 |
| VM22 (Proxmox) | 10.84.119.203 | Fixe | https://10.84.119.203:8006 |

Les machines Proxmox ont une IP fixe, définie dans `/etc/network/interfaces` et `/etc/hosts`. Les machines Ubuntu reçoivent leur adresse par DHCP : elles peuvent changer à chaque connexion, donc je les relève avec `ip a` avant chaque test.

En complément, les machines Ubuntu (VM2, VM11 et VM12) disposent de **Cockpit**, accessible à l'adresse `https://IP_DE_LA_MACHINE:9090` (par exemple `https://10.84.119.81:9090` pour VM2), avec l'utilisateur Ubuntu de la machine. Sur VM2, l'onglet *Virtual Machines* affiche vm21 et vm22.

### 5.3 Comptes d'accès

- Proxmox (VM1, VM21, VM22) : utilisateur `root`, realm *Linux PAM*. Sur téléphone, la liste du realm est vide : il faut saisir `root@pam`.
- Ubuntu : un utilisateur par machine (`vm2`, `vm11`, `vm12`).
- Les mots de passe sont connus uniquement de moi et ne figurent pas dans ce rapport.

## 6. Déroulement de la réalisation

1. **Préparation du PC.** Vérification de la virtualisation (VT-x activé), libération de l'espace disque, désinstallation du SDK Windows inutile et suppression d'une distribution Ubuntu WSL qui occupait l'hyperviseur de Windows.
2. **Création de VM1.** Nouvelle machine VMware (Debian 11 64 bits, 2 vCPU, 4 Go, 25 Go), activation de *Virtualize Intel VT-x/EPT*, puis installation de Proxmox VE 7.4 depuis l'ISO.
3. **Vérification de l'imbriqué sur VM1.** La commande `egrep -c '(vmx|svm)' /proc/cpuinfo` a renvoyé 4, ce qui prouve que les instructions de virtualisation sont visibles dans VM1.
4. **VM11 et VM12.** Envoi de l'ISO Ubuntu Server dans le stockage de Proxmox, puis création des deux machines depuis l'interface web (1 coeur, 1 Go, 10 Go) et installation d'Ubuntu.
5. **Création de VM2.** Installation d'Ubuntu Server, ajout de Xubuntu, puis de KVM et de virt-manager. La commande `kvm-ok` a confirmé : *KVM acceleration can be used*.
6. **Pont réseau dans VM2.** Création du pont `br0` avec Netplan afin que VM21 et VM22 aient accès au même réseau que les autres machines.
7. **VM21 et VM22.** Copie de l'ISO Proxmox dans VM2 avec `scp`, création des machines dans virt-manager (host-passthrough, réseau sur `br0`) et installation de Proxmox.
8. **Passage en bridge sur le point d'accès.** VMnet0 associé à la carte Wi-Fi, adaptateurs de VM1 et VM2 en Bridged, puis modification des adresses IP fixes des trois Proxmox.
9. **Services web.** Installation d'Apache2 sur VM2, VM11 et VM12, avec une page indiquant le nom de la machine.
10. **Cockpit.** Installation du paquet `cockpit` et activation du service avec `systemctl enable --now cockpit.socket` sur VM2, VM11 et VM12, puis ajout de `cockpit-machines` sur VM2 pour gérer les machines KVM depuis le navigateur. L'interface est disponible sur le port 9090.

Exemple de configuration du pont Netplan sur VM2 (`/etc/netplan/00-installer-config.yaml`) :

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: false
  bridges:
    br0:
      interfaces:
        - ens33
      dhcp4: true
```

## 7. Tests et vérifications

| Test | Résultat |
|---|---|
| `egrep -c '(vmx\|svm)' /proc/cpuinfo` sur VM1 | 4 : virtualisation imbriquée disponible |
| `kvm-ok` sur VM2 | KVM acceleration can be used |
| `ping -c 4 8.8.8.8` sur VM11, VM12 et VM2 | 4 paquets reçus sur 4, aucune perte |
| `ping google.com` sur VM11 et VM12 | Résolution DNS et accès Internet corrects |
| Interface web de VM1, VM21, VM22 depuis Windows | Page de connexion Proxmox affichée |
| Interface web de VM22 depuis le téléphone | Page de connexion affichée sur 10.84.119.203:8006 |
| Pages web de VM2, VM11, VM12 | Le nom de la machine s'affiche |

Ces résultats montrent que les six machines sont joignables sur le même réseau et que la virtualisation imbriquée fonctionne.

### Captures d'écran

<!-- Enlevez les balises de commentaire quand vous avez ajouté les images dans le dossier images/ -->
<!-- ![kvm-ok sur VM2](images/kvm-ok.png) -->
<!-- ![Interface Proxmox de VM1 avec VM11 et VM12](images/proxmox-vm1.png) -->
<!-- ![virt-manager avec vm21 et vm22](images/virt-manager.png) -->
<!-- ![Ping depuis VM11](images/ping-vm11.png) -->
<!-- ![Page de connexion de VM22 sur téléphone](images/vm22-telephone.png) -->

## 8. Problèmes rencontrés et solutions

**1. Virtualisation imbriquée refusée (message *VT-x/EPT is not supported*).** Windows utilisait déjà son propre hyperviseur (WSL 2, plateforme de machine virtuelle, intégrité de la mémoire). Solution : désactiver ces fonctionnalités dans `optionalfeatures`, désactiver l'intégrité de la mémoire et exécuter `bcdedit /set hypervisorlaunchtype off`, puis redémarrer. Le message est revenu plus tard, et j'ai dû répéter la manipulation.

**2. Espace disque et mémoire limités.** Avec seulement 50 Go libres et 16 Go de RAM, les six machines devaient rester légères. J'ai utilisé des disques dynamiques, réduit la RAM des machines de niveau 2 et éteint la machine vCSA d'un autre TP, qui monopolisait 13,6 Go.

**3. Clavier de la console Proxmox en QWERTY.** Les touches ne correspondaient pas à mon clavier AZERTY. Solution : `dpkg-reconfigure keyboard-configuration` pour choisir le clavier français. Les chiffres demandent aussi la touche Maj, ce qui a provoqué plusieurs erreurs de saisie (par exemple *invalid argument* lors d'un `ping`).

**4. Inversion de l'architecture.** J'avais d'abord installé un Proxmox dans Proxmox. En relisant le schéma, j'ai compris que VM11 et VM12 appartiennent à VM1 et que VM21 et VM22 appartiennent à VM2. J'ai supprimé la machine concernée et recommencé.

**5. Échec de l'installation de Proxmox dans VM21.** Avec un disque de 10 Go, l'installateur a affiché *Size is too big* et *unable to create data volume*. Proxmox réserve de l'espace pour le swap et le système, et il ne restait rien pour les données. Solution : disque de 20 Go.

**6. Erreur de syntaxe YAML pour le pont Netplan.** Le collage dans l'éditeur a décalé les espaces, et Netplan a refusé le fichier (*inconsistent indentation*). Solution : écrire le fichier avec la commande `tee` et un bloc `EOF`, qui conserve l'indentation. La session SSH s'est coupée au passage de l'adresse IP sur `br0` ; j'ai terminé depuis la console de VMware avec `netplan apply`, puis `systemctl restart systemd-networkd`.

**7. Copie de l'ISO par SSH.** Une première tentative a échoué car le délai d'authentification SSH (2 minutes) était dépassé pendant la saisie. Une erreur de commande `mv` a aussi renommé l'ISO en `ls`, et l'ISO a été supprimée avec une machine virtuelle. Je l'ai recopiée depuis Windows et j'ai appris à terminer un dossier cible par une barre oblique.

**8. États sauvegardés.** Une restauration de VM1 a échoué (*features supported by the processors are different*), et vm21 et vm22 apparaissaient comme *Sauvegardé*. Solution : abandonner l'état avec *Discard* ou `virsh managedsave-remove`, puis éteindre les machines proprement avec `shutdown -h now` plutôt que de les suspendre.

**9. Changement de réseau.** Les machines étaient d'abord en NAT (192.168.74.0/24), et ce réseau n'était pas visible depuis le point d'accès. J'ai lié VMnet0 à la carte Wi-Fi, mis les adaptateurs en Bridged et modifié l'adresse IP, la passerelle et le fichier `/etc/hosts` des machines Proxmox.

**10. Coupure du Wi-Fi pendant les installations.** Les téléchargements d'`apt` ont échoué (*Network is unreachable*). Après reconnexion, l'installation a repris normalement.

**11. Connexion Proxmox sur téléphone.** La liste du realm était vide. Solution : saisir `root@pam` comme nom d'utilisateur.

## 9. Limites et précautions

- Les adresses des machines Ubuntu (DHCP) peuvent changer ; les adresses des machines Proxmox sont fixes mais ne fonctionnent que sur le réseau 10.84.119.0/24.
- Si un autre appareil prend l'une de mes adresses fixes quand mes machines sont éteintes, un conflit d'adresse peut apparaître.
- Le mode bridge sur une carte Wi-Fi est moins fiable qu'en Ethernet, et certains points d'accès isolent les appareils entre eux.
- Avec 16 Go de RAM, je ne peux pas faire tourner les six machines avec des ressources confortables ; je dois fermer les autres applications et garder le chargeur branché.
- Pour éviter la corruption des machines, il faut les éteindre proprement, dans l'ordre : machines de niveau 2, puis machines de niveau 1, puis Windows.
- Cockpit ne doit pas servir à modifier le réseau de VM2 : les ponts sont gérés par Netplan et une modification depuis l'interface pourrait les casser. Le port 9090 doit être autorisé dans le pare-feu (`ufw`) s'il est actif.

## 10. Conclusion

Ce TP m'a permis de mettre en place une architecture de virtualisation sur deux niveaux, avec trois hyperviseurs différents : VMware Workstation, Proxmox VE et KVM/QEMU. J'ai compris l'importance de la virtualisation matérielle imbriquée et ses conflits avec l'hyperviseur de Windows. J'ai aussi appris à gérer le réseau en mode pont (VMware, Proxmox et Ubuntu), à dimensionner des ressources limitées et à diagnostiquer des erreurs de configuration. Les six machines sont opérationnelles, ont accès à Internet et sont joignables par leur adresse IP depuis un autre appareil connecté au même point d'accès.
