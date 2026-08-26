# deep-in-system — Administration d'un serveur Ubuntu

**Auteur :** `clecart` — **Hostname :** `clecart-host`

Ce document est à la fois le **compte-rendu** du projet et la **procédure complète** :
chaque commande y est expliquée, dans l'ordre où elle doit être exécutée.
Il permet de reconstruire le serveur entièrement depuis zéro.

---

## Sommaire

0. [Vue d'ensemble et choix d'architecture](#0-vue-densemble-et-choix-darchitecture)
1. [Création de la VM et partitionnement](#1-création-de-la-vm-et-partitionnement)
2. [Première connexion, mise à jour, hostname et clavier](#2-première-connexion-mise-à-jour-hostname-et-clavier)
3. [Réseau : IP statique, zéro DHCP](#3-réseau--ip-statique-zéro-dhcp)
4. [SSH : port 2222, root interdit](#4-ssh--port-2222-root-interdit)
5. [Pare-feu UFW](#5-pare-feu-ufw)
6. [Utilisateurs : luffy et zoro](#6-utilisateurs--luffy-et-zoro)
7. [Serveur FTP : vsftpd et l'utilisateur nami](#7-serveur-ftp--vsftpd-et-lutilisateur-nami)
8. [Base de données MySQL](#8-base-de-données-mysql)
9. [WordPress](#9-wordpress)
10. [Sauvegarde automatique par cron](#10-sauvegarde-automatique-par-cron)
11. [Bonus](#11-bonus)
12. [Export OVA, sha1 et rendu](#12-export-ova-sha1-et-rendu)
13. [Récapitulatif des ports ouverts (justification audit)](#13-récapitulatif-des-ports-ouverts-justification-audit)
14. [Mémo audit : créer l'utilisateur kratos en moins de 10 minutes](#14-mémo-audit--créer-lutilisateur-kratos-en-moins-de-10-minutes)
15. [Checklist finale avant audit](#15-checklist-finale-avant-audit)
16. [Questions d'audit — réponses types](#16-questions-daudit--réponses-types)
17. [Grille d'audit officielle — commandes et sorties attendues](#17-grille-daudit-officielle--commandes-et-sorties-attendues)
18. [Glossaire des commandes](#18-glossaire-des-commandes)

---

## 0. Vue d'ensemble et choix d'architecture

### 0.1 Ce que le serveur doit faire

| Brique | Logiciel | Rôle |
|---|---|---|
| Système | Ubuntu Server LTS | Base du serveur, sans interface graphique |
| Réseau | `netplan` + `systemd-networkd` | IP statique, aucune interface en DHCP |
| Accès distant | OpenSSH sur le port `2222` | Administration à distance, root interdit |
| Pare-feu | `ufw` (front-end de `iptables`/`nftables`) | Tout fermé en entrée sauf ce qui est justifié |
| Transfert de fichiers | `vsftpd` | Utilisateur `nami`, lecture seule sur `/backup` |
| Base de données | MySQL Server 8 | Écoute uniquement sur `127.0.0.1` |
| Serveur web | Apache 2 + PHP | Sert WordPress à la racine du site |
| CMS | WordPress | `http://{host}/` |
| Sauvegarde | script bash + `cron` | Dump de la base tous les jours à 00:00 |

### 0.2 Choix d'architecture et justifications

**Démarrage en BIOS (EFI désactivé)**
En UEFI, Ubuntu impose une partition système EFI (`/boot/efi`) d'environ **1 Go**.
Or sur un disque de 30 Go, `swap 4G + / 15G + /home 5G + /backup 6G` fait
**déjà 30 Go** : il ne reste pas la place pour l'ESP. En BIOS, GRUB n'a besoin
que d'une minuscule partition `bios_grub` de **1 Mio** (créée automatiquement
par l'installeur) pour y loger son deuxième étage. Les 4 partitions demandées
tiennent donc quasiment à l'octet près — `/backup` récupère simplement le reste,
soit ~5,99 Go au lieu de 6,00 Go exactement.

**Deux cartes réseau, les deux en IP statique**

| Carte | Mode VirtualBox | IP fixe | Rôle |
|---|---|---|---|
| `enp0s3` | NAT | `10.0.2.15/24`, passerelle `10.0.2.2` | Accès Internet (`apt`, téléchargement de WordPress) |
| `enp0s8` | Réseau privé hôte (host-only) | `192.168.56.10/24`, pas de passerelle | SSH / HTTP / FTP depuis la machine hôte |

Pourquoi pas du **bridge** ? Une IP bridgée dépend du réseau physique où tourne
la VM. À l'audit, la VM est relancée sur **une autre machine, sur un autre
réseau** : l'IP entrerait en conflit ou ne serait plus joignable. Le couple
NAT + host-only est identique partout et ne dépend d'aucun routeur externe.

Point de vigilance sur le NAT : le réseau `10.0.2.0/24` de VirtualBox est
**figé** (passerelle `10.0.2.2`, DNS `10.0.2.3`, premier client `10.0.2.15`).
On peut donc écrire ces valeurs en dur en statique — elles seront valides sur
n'importe quelle machine d'audit.

**Aucune passerelle sur la carte host-only** : deux routes par défaut
provoqueraient un routage aléatoire. Une seule route par défaut, via le NAT.

**Pas de LVM** : le sujet demande 4 partitions fixes, jamais redimensionnées.
LVM ajouterait une couche d'abstraction (PV → VG → LV) à expliquer sans aucun
bénéfice ici. On reste sur des partitions classiques, plus simples à auditer
avec `lsblk`.

### 0.3 Valeurs retenues

Toutes les valeurs concrètes de cette installation, rassemblées ici pour servir
d'aide-mémoire le jour de l'audit.

| Élément | Valeur |
|---|---|
| Système | Ubuntu Server 26.04 LTS (*resolute*) |
| Nom de la VM | `deep-in-system` |
| Hostname | `clecart-host` |
| Utilisateur principal | `clecart` (groupe `sudo`) |
| Interface NAT | `enp0s3` — `10.0.2.15/24`, passerelle `10.0.2.2` |
| Interface host-only | `enp0s8` — `192.168.56.10/24`, sans passerelle |
| DNS | `8.8.8.8`, `1.1.1.1` |
| Port SSH | `2222` |
| Clé SSH du projet | `~/.ssh/deep_in_system` sur le poste, installée pour `luffy` |
| Utilisateurs créés | `luffy` (sudo, clé), `zoro` (mot de passe), `nami` (FTP seul) |
| Base de données | `wordpress`, utilisateur `wp_user`@`localhost` |
| Titre du site | `deep-in-system` |
| Administrateur WordPress | `clecart` |
| Répertoire de sauvegarde | `/backup` (`750 root:nami`) |
| Journal de sauvegarde | `/var/log/backup.log` (`644`) |

> Les mots de passe ne figurent volontairement **pas** dans ce fichier : il est
> versionné et poussé sur le dépôt Git. Les conserver hors du dépôt.

---

## 1. Création de la VM et partitionnement

### 1.1 Télécharger l'ISO

Récupérer la **dernière LTS** de Ubuntu Server sur <https://ubuntu.com/download/server>.
Les LTS sont supportées 5 ans, contrairement aux versions intermédiaires supportées
9 mois — c'est le seul choix raisonnable pour un serveur, et la grille d'audit
l'exige explicitement.

**Version utilisée ici : Ubuntu Server 26.04 LTS** (nom de code *resolute*),
fichier `ubuntu-26.04-live-server-amd64.iso`, environ 2,8 Go.

Vérifier l'intégrité du téléchargement (bonne pratique, même logique que le
`sha1sum` demandé au rendu) :

```bash
sha256sum ubuntu-*-live-server-amd64.iso
# comparer avec le fichier SHA256SUMS publié sur le site d'Ubuntu
```

### 1.2 Créer la machine virtuelle

Dans VirtualBox → **Nouvelle** :

| Paramètre | Valeur |
|---|---|
| Nom | `deep-in-system` |
| Type / Version | Linux / Ubuntu (64-bit) |
| RAM | 2048 Mo minimum (4096 Mo confortable) |
| CPU | 2 vCPU |
| Disque dur | **30 Go**, VDI, alloué dynamiquement |
| EFI | **décoché** (Système → Activer l'EFI : non) |

Puis **Configuration → Réseau** :

- **Carte 1** : Activée, mode **NAT**
- **Carte 2** : Activée, mode **Réseau privé hôte**, `vboxnet0`

Toujours dans VirtualBox : **Outils → Gestionnaire de réseau → vboxnet0 →
décocher « Serveur DHCP »**. Le sujet interdit toute interface en attribution
dynamique ; couper le serveur DHCP côté hôte garantit qu'aucune IP ne peut être
distribuée par accident.

Enfin, **Stockage → lecteur optique → choisir l'ISO** téléchargée, puis démarrer.

### 1.3 Installation d'Ubuntu Server

Écrans successifs de l'installeur (`subiquity`) :

| Écran | Réponse |
|---|---|
| Langue / clavier | English / French (AZERTY) |
| Type d'installation | **Ubuntu Server** (pas « minimized ») |
| Réseau | Laisser tel quel — on reconfigurera en statique après |
| Proxy / miroir | Vide / valeur par défaut |
| Guided storage | **Custom storage layout** ← indispensable |
| Profile | Nom : `clecart`, **hostname : `clecart-host`**, user : `clecart` |
| Ubuntu Pro | Skip |
| OpenSSH Server | **Install OpenSSH server : coché** |
| Snaps | Aucun |

### 1.4 Le partitionnement manuel

Dans **Custom storage layout**, sélectionner le disque `/dev/sda` → **Use As
Boot Device**. Cela crée automatiquement une petite partition `bios_grub` de 1 Mo
(zone réservée à GRUB en mode BIOS/GPT) — c'est normal et invisible ensuite.

Créer ensuite 4 partitions avec **Add GPT Partition** :

| Ordre | Taille | Format | Point de montage | Rôle |
|---|---|---|---|---|
| 1 | `4G` | swap | — | Mémoire virtuelle sur disque |
| 2 | `15G` | ext4 | `/` | Système : noyau, binaires, config, logs |
| 3 | `5G` | ext4 | `/home` | Données des utilisateurs |
| 4 | reste (`~6G`) | ext4 | `/backup` | Archives de sauvegarde |

**Pourquoi séparer les partitions ?**

- **`swap` (4G)** : espace disque utilisé quand la RAM est saturée. Le noyau y
  déplace les pages mémoire inactives. Règle usuelle : 1× à 2× la RAM.
- **`/home` séparé** : un utilisateur qui remplit son répertoire personnel
  (upload, logs, téléchargement) ne peut **pas** saturer `/`. Un `/` plein rend
  le système inutilisable (impossible d'écrire un log, de créer un fichier
  temporaire, parfois de se connecter). C'est la principale raison de la
  séparation. Elle permet aussi de réinstaller le système sans toucher aux
  données utilisateurs.
- **`/backup` séparé** : même logique, et c'est ce qui donne du sens à la
  sauvegarde — les archives ne vivent pas sur le même système de fichiers que
  les données d'origine. Une corruption du FS de `/` ne détruit pas les backups.

Valider avec **Done → Continue** (l'installeur avertit que les données du disque
vont être effacées : c'est un disque virtuel neuf, on confirme).

À la fin : **Reboot Now**, retirer l'ISO du lecteur optique si VirtualBox ne le
fait pas seul.

> ⚠️ **Défaut connu de l'installeur** : après avoir tout écrit sur le disque,
> l'environnement live se démonte lui-même puis n'arrive plus à exécuter son
> binaire d'extinction. L'écran affiche alors :
>
> ```
> [!!!!!!] Failed to execute shutdown binary.
> ```
>
> **L'installation sur le disque est intacte** — seul le redémarrage échoue. Il
> suffit d'éteindre la VM de force et de la relancer :
>
> ```bash
> VBoxManage controlvm deep-in-system poweroff
> VBoxManage modifyvm  deep-in-system --boot1 disk --boot2 dvd
> VBoxManage startvm   deep-in-system --type gui
> ```
>
> Éjecter aussi le disque du lecteur, sinon la VM redémarre sur l'installeur :
> `VBoxManage storageattach deep-in-system --storagectl SATA --port 1 --device 0 --type dvddrive --medium emptydrive --forceunmount`

### 1.5 Vérifier le partitionnement

```bash
lsblk -f
```

`lsblk` (list block devices) affiche l'arborescence des périphériques bloc ;
l'option `-f` ajoute le système de fichiers, l'UUID et le point de montage.
Résultat attendu : 4 partitions, dont une `swap` et trois `ext4` montées sur
`/`, `/home` et `/backup`.

```bash
df -h
```

`df` (disk free) montre l'espace **utilisé et disponible par point de montage**,
`-h` pour un affichage lisible (Go/Mo). C'est la commande à montrer à l'audit :
elle prouve que `/`, `/home` et `/backup` sont bien des systèmes de fichiers
distincts.

```bash
free -h
swapon --show
```

`free -h` affiche l'usage RAM et swap ; `swapon --show` liste les zones de swap
actives et confirme les 4 Go.

---

## 2. Première connexion, mise à jour, hostname et clavier

### 2.1 Mise à jour du système

```bash
sudo apt update && sudo apt upgrade -y
```

- `apt update` : télécharge la **liste** des paquets disponibles depuis les
  dépôts déclarés dans `/etc/apt/sources.list`. Ne met rien à jour.
- `apt upgrade` : installe effectivement les nouvelles versions des paquets
  déjà présents. `-y` répond « oui » automatiquement.
- `&&` : n'exécute la seconde commande que si la première a réussi (code de
  retour 0). Inutile de tenter une mise à jour si la liste n'a pas pu être
  rafraîchie.

`sudo` exécute la commande avec les privilèges de root **pour cette commande
seulement**. C'est tout l'intérêt par rapport à `su -` : on sait exactement quel
programme tourne en privilégié, et chaque appel est tracé dans
`/var/log/auth.log`.

### 2.2 Vérifier le hostname

Il a été défini à l'installation, mais on vérifie :

```bash
hostnamectl
```

Si besoin de le corriger :

```bash
sudo hostnamectl set-hostname clecart-host
```

`hostnamectl` est l'outil `systemd` de gestion du nom de machine ; il écrit dans
`/etc/hostname` et applique le changement à chaud, sans redémarrage.

Il faut ensuite s'assurer que le nom se résout localement, sinon `sudo` peut
mettre plusieurs secondes à répondre (il tente de résoudre le nom de la machine) :

```bash
sudo nano /etc/hosts
```

Le fichier doit contenir :

```
127.0.0.1   localhost
127.0.1.1   clecart-host
```

Vérification :

```bash
hostname          # -> clecart-host
ping -c 1 clecart-host
```

### 2.3 Sauvegarder les fichiers de configuration

Réflexe à prendre **avant chaque modification** : on garde l'original.

```bash
sudo mkdir -p /root/config-backup
sudo cp /etc/ssh/sshd_config /root/config-backup/sshd_config.orig
```

Convention utilisée dans tout ce document : `.orig` = version d'origine jamais
modifiée. En cas de service cassé, on restaure et on repart de zéro.

### 2.4 Disposition du clavier de la console

La disposition choisie à l'écran d'accueil de l'installeur s'applique à la
**console de la VM**. Si elle ne correspond pas au clavier physique, les touches
ne produisent pas les caractères attendus.

**Vérifier la configuration en vigueur :**

```bash
cat /etc/default/keyboard
```

**La passer en français :**

```bash
sudo sed -i 's/^XKBLAYOUT=.*/XKBLAYOUT="fr"/' /etc/default/keyboard
sudo setupcon
sudo loadkeys fr
```

Trois commandes, trois rôles distincts qu'il faut savoir distinguer :

| Commande | Effet | Persistant ? |
|---|---|---|
| Édition de `/etc/default/keyboard` | Déclare la disposition voulue | **Oui** — c'est le seul fichier de référence |
| `setupcon` | Applique le fichier aux consoles | Non, mais rejoué au démarrage par `console-setup.service` |
| `loadkeys fr` | Charge la table dans le noyau, immédiatement | **Non** — perdu au redémarrage |

> ⚠️ **`localectl` n'existe pas** sur une installation Ubuntu Server : il est
> fourni par le paquet `systemd` mais absent de l'image serveur. La commande
> échoue avec `command not found`. La configuration clavier passe donc
> obligatoirement par `/etc/default/keyboard`.

**Vérifier le résultat** — la table réellement chargée dans le noyau :

```bash
sudo dumpkeys | grep -w "keycode  16"
```

Le *keycode* 16 est la touche située en haut à gauche de la rangée alphabétique :
elle donne `q` en QWERTY et `a` en AZERTY. La sortie doit afficher `keycode 16 = +a`.

Vérifier aussi que la disposition sera rejouée à chaque démarrage :

```bash
systemctl is-enabled console-setup.service    # -> enabled
```

### ⚠️ Le piège du mot de passe saisi dans la mauvaise disposition

Sur un clavier français, **les chiffres s'obtiennent avec `Shift`** : la rangée
du haut donne `&é"'(-è_çà` sans modificateur. Sur une console en QWERTY, ces
mêmes touches physiques donnent directement les chiffres, et `Shift` donne la
ponctuation.

Conséquence : un mot de passe saisi à l'installation en croyant taper
`Rouen76Serveur` sur une console QWERTY produit en réalité **`Rouen&^Serveur`**
— `Shift+7` donne `&`, `Shift+6` donne `^`. Le mot de passe enregistré n'est pas
celui qu'on croit, et l'erreur ne se révèle qu'à la première connexion depuis
une autre machine.

Deux protections :

1. **Choisir un mot de passe sans caractère ambigu** : uniquement des lettres et
   des chiffres, en évitant `a`, `q`, `z`, `w`, `m` et `y`, qui changent de
   position entre AZERTY, QWERTY et QWERTZ. Toutes les autres lettres tombent sur
   la même touche physique dans les trois dispositions.
2. **En cas de doute, réécrire le mot de passe sans passer par un clavier** :

```bash
echo 'clecart:NouveauMotDePasse' | sudo chpasswd
```

`chpasswd` lit des couples `utilisateur:mot_de_passe` sur l'entrée standard et
écrit directement le condensat dans `/etc/shadow`. Aucune disposition clavier
n'intervient : ce qui est écrit est exactement ce qui sera attendu.

---

## 3. Réseau : IP statique, zéro DHCP

### 3.1 Identifier les interfaces

```bash
ip -brief address
ip -brief link
```

`ip` remplace les anciennes commandes `ifconfig` / `route`. `-brief` donne une
sortie tabulaire lisible : nom de l'interface, état (UP/DOWN), adresses.

Sortie typique dans VirtualBox :

```
lo      UNKNOWN  127.0.0.1/8
enp0s3  UP       10.0.2.15/24     <- carte NAT
enp0s8  UP       192.168.56.x/24  <- carte host-only
```

> Les noms peuvent varier (`enp0s17`, `enp0s8`…). **Utiliser les noms réellement
> affichés** dans les fichiers ci-dessous.

Le nommage `enp0s3` est le *Predictable Network Interface Naming* de systemd :
`en` = ethernet, `p0` = bus PCI 0, `s3` = slot 3. Contrairement à l'ancien
`eth0`, ce nom ne change pas selon l'ordre de détection au démarrage — ce qui
est indispensable quand on écrit une configuration statique.

### 3.2 Neutraliser cloud-init

L'image serveur d'Ubuntu embarque `cloud-init`, qui **réécrit la configuration
réseau à chaque démarrage** (typiquement en DHCP). Si on ne le désactive pas, la
configuration statique est perdue au reboot — erreur classique à l'audit.

```bash
echo 'network: {config: disabled}' | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

- `echo` écrit la chaîne sur la sortie standard.
- `|` (pipe) redirige cette sortie vers l'entrée de la commande suivante.
- `sudo tee fichier` écrit ce qu'il reçoit dans le fichier **avec les droits
  root**. On ne peut pas écrire `sudo echo ... > fichier` : la redirection `>`
  est faite par le shell, qui lui n'est pas root.

### 3.3 Écrire la configuration statique

On écarte le fichier généré par l'installeur (netplan lit **tous** les `*.yaml`
du répertoire ; renommer l'extension suffit à l'ignorer) :

```bash
sudo mkdir -p /root/config-backup
sudo cp /etc/netplan/00-installer-config.yaml /root/config-backup/00-installer-config.yaml.orig
sudo mv /etc/netplan/00-installer-config.yaml /root/config-backup/00-installer-config.yaml.disabled
```

> Le nom de ce fichier varie selon la version : `00-installer-config.yaml` sur
> 26.04, `50-cloud-init.yaml` sur les versions antérieures. Vérifier avec
> `ls /etc/netplan/` plutôt que de supposer.

Puis on crée notre fichier :

```bash
sudo nano /etc/netplan/01-static-config.yaml
```

Contenu (⚠️ YAML : **indentation en espaces uniquement, jamais de tabulation**) :

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:                       # carte NAT -> Internet
      dhcp4: false
      dhcp6: false
      accept-ra: false
      addresses:
        - 10.0.2.15/24
      routes:
        - to: default
          via: 10.0.2.2
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
    enp0s8:                       # carte host-only -> accès depuis l'hôte
      dhcp4: false
      dhcp6: false
      accept-ra: false
      addresses:
        - 192.168.56.10/24
```

Explication ligne à ligne :

| Clé | Signification |
|---|---|
| `version: 2` | Version du format netplan (la seule utilisée aujourd'hui) |
| `renderer: networkd` | netplan n'est qu'un traducteur : il génère la conf de `systemd-networkd` (le gestionnaire réseau sans interface graphique). L'alternative `NetworkManager` est destinée aux postes de bureau |
| `dhcp4/dhcp6: false` | **Exigence du sujet** : aucune attribution dynamique, en IPv4 comme en IPv6 |
| `accept-ra: false` | Refuse les *Router Advertisements* IPv6. Sans cette ligne, l'auto-configuration IPv6 (SLAAC) peut attribuer une adresse marquée `dynamic` **alors même que `dhcp6` est à `false`** — et la commande d'audit `ip a \| grep dynamic` ne renverrait pas une sortie vide |
| `addresses` | IP + masque en notation CIDR. `/24` = `255.255.255.0`, soit 254 hôtes utilisables |
| `routes: to: default via:` | Route par défaut : tout paquet dont la destination n'est pas sur le réseau local part vers cette passerelle. Une seule interface doit en avoir une |
| `nameservers` | Serveurs DNS interrogés pour convertir un nom (`google.com`) en IP |

**Pourquoi `10.0.2.15` et `10.0.2.2` ?** Le réseau NAT de VirtualBox est câblé
en dur : le routeur virtuel est toujours en `.2`, le serveur DNS en `.3`, et la
première IP distribuée est `.15`. En les figeant en statique, on conserve un
accès Internet fonctionnel **sur n'importe quelle machine hôte**.

Ces adresses sont **privées** au sens de la RFC 1918 (`10.0.0.0/8` et
`192.168.0.0/16`) : elles ne sont pas routables sur Internet, ce qui est bien ce
que demande le sujet.

### 3.4 Appliquer et sécuriser

```bash
sudo chmod 600 /etc/netplan/01-static-config.yaml
sudo netplan generate
sudo netplan apply
```

- `chmod 600` : lecture/écriture pour root uniquement. Netplan émet un
  avertissement si le fichier est lisible par tous (il peut contenir des clés
  Wi-Fi).
- `netplan generate` : traduit le YAML en fichiers `systemd-networkd` et
  **valide la syntaxe** sans rien appliquer. À faire toujours en premier.
- `netplan apply` : applique la configuration à chaud.

> En cas de doute sur une conf distante, `sudo netplan try` applique la
> configuration et **revient automatiquement en arrière au bout de 120 s** si on
> ne confirme pas — filet de sécurité contre la perte de la session SSH.

### 3.5 Vérifications

```bash
ip -brief address                    # les IP fixes sont bien là
ip route                             # une seule route par défaut, via 10.0.2.2
ping -c 5 10.0.2.2                   # la passerelle répond
ping -c 5 8.8.8.8                    # le routage IP fonctionne
ping -c 5 google.com                 # la résolution DNS fonctionne
resolvectl status | grep -i "DNS Servers"
```

Distinguer `ping 8.8.8.8` et `ping google.com` est important : si le premier
passe et pas le second, le problème est **uniquement DNS**, pas réseau.

Preuve qu'aucune interface n'est en DHCP (à montrer à l'audit) :

```bash
ip a | grep dynamic                  # -> AUCUNE sortie : c'est LA commande de l'audit
grep -ri dhcp /etc/netplan/          # -> uniquement "dhcp4: false" / "dhcp6: false"
ps aux | grep -c "[d]hclient"        # -> 0 : aucun client DHCP en cours
networkctl status                    # état des interfaces vu par systemd-networkd
```

> `ip a | grep dynamic` doit renvoyer **exactement rien**. Le mot-clé `dynamic`
> apparaît sur toute adresse obtenue automatiquement, en IPv4 **comme en IPv6** :
> c'est le test formel de l'audit sur l'absence de DHCP.

Enfin, on redémarre et on revérifie — c'est le seul moyen de prouver que
cloud-init ne réécrit plus rien :

```bash
sudo reboot
# puis, après reconnexion :
ip -brief address
```

---

## 4. SSH : port 2222, root interdit

SSH (*Secure SHell*) chiffre la session d'administration distante. Le service
côté serveur s'appelle `sshd` (*SSH daemon*).

### 4.1 Le piège d'Ubuntu 24.04 : l'activation par socket

Depuis Ubuntu 24.04, SSH est démarré **par socket systemd** (`ssh.socket`) :
c'est `systemd` qui écoute sur le port et lance `sshd` à la connexion. Dans ce
mode, **la directive `Port` de `sshd_config` est ignorée** — le port est défini
dans l'unité socket. On repasse en démarrage classique :

```bash
systemctl is-enabled ssh.socket        # si "enabled", il faut la désactiver
sudo systemctl disable --now ssh.socket
sudo systemctl enable --now ssh.service
```

- `systemctl` pilote les services de `systemd` (le gestionnaire de services,
  PID 1, premier processus lancé par le noyau).
- `disable` : ne démarre plus au boot. `--now` : agit aussi immédiatement.
- `enable --now` : démarre maintenant **et** à chaque démarrage.

### 4.2 Configurer sshd

```bash
sudo cp /etc/ssh/sshd_config /root/config-backup/sshd_config.orig
sudo nano /etc/ssh/sshd_config
```

Directives à définir :

```
Port 2222
PermitRootLogin no
PasswordAuthentication yes
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 3
LoginGraceTime 30
X11Forwarding no

# luffy s'authentifie exclusivement par clé
Match User luffy
    PasswordAuthentication no
```

| Directive | Rôle |
|---|---|
| `Port 2222` | **Exigence du sujet**. Déplacer le port n'est pas de la sécurité au sens strict (*security through obscurity*), mais cela élimine l'écrasante majorité des scans automatisés qui ne testent que le 22 |
| `PermitRootLogin no` | **Exigence du sujet**. `root` est le seul compte dont le nom est connu d'avance sur *toutes* les machines Linux : l'interdire supprime la moitié du travail d'un attaquant (deviner l'identifiant). On passe par un compte nominatif + `sudo`, ce qui rend chaque action traçable |
| `PasswordAuthentication yes` | Nécessaire pour `zoro`, dont le sujet impose l'authentification par mot de passe |
| `PubkeyAuthentication yes` | Authentification par paire de clés, nécessaire pour `luffy` |
| `PermitEmptyPasswords no` | Interdit tout compte sans mot de passe |
| `MaxAuthTries 3` | Ferme la connexion après 3 échecs : ralentit le *brute force* |
| `LoginGraceTime 30` | Délai maximal pour s'authentifier ; limite les connexions ouvertes non authentifiées |
| `X11Forwarding no` | Aucune application graphique sur un serveur : on supprime la fonctionnalité, donc sa surface d'attaque |
| `Match User luffy` | Bloc conditionnel : les directives qui suivent (indentées) ne s'appliquent **qu'à cet utilisateur**. `luffy` ne pourra donc se connecter que par clé |

> ⚠️ Un bloc `Match` s'applique à **tout ce qui le suit** jusqu'au prochain
> `Match` ou jusqu'à la fin du fichier. Il doit donc impérativement être placé
> **en dernier**, après toutes les directives globales, sinon celles-ci se
> retrouvent enfermées dans la condition.

**Attention aux fichiers d'inclusion.** La première ligne de `sshd_config` est
`Include /etc/ssh/sshd_config.d/*.conf`, et en SSH **la première valeur lue
gagne**. Un fichier déposé là (par exemple `50-cloud-init.conf`) écrase donc
silencieusement notre configuration. À vérifier :

```bash
ls -l /etc/ssh/sshd_config.d/
sudo cat /etc/ssh/sshd_config.d/*.conf 2>/dev/null
```

Si un fichier y contredit nos réglages, le déplacer dans `/root/config-backup/`.

### 4.3 Valider avant de redémarrer le service

```bash
sudo sshd -t
```

`-t` = *test mode* : contrôle la syntaxe **sans** appliquer. Une erreur ici et
le service refusera de redémarrer — se retrouver sans SSH sur un serveur
distant est le scénario à éviter absolument.

```bash
sudo systemctl restart ssh
sudo systemctl status ssh
```

Vérifier la configuration **effective** (celle que `sshd` applique réellement,
inclusions comprises) :

```bash
sudo sshd -T | grep -Ei "^(port|permitrootlogin|passwordauthentication|pubkeyauthentication)"
```

Et le port réellement en écoute :

```bash
sudo ss -tlnp | grep sshd
```

`ss` (*socket statistics*) remplace `netstat` : `-t` TCP, `-l` sockets en écoute
(*listening*), `-n` ports numériques, `-p` processus propriétaire.
Résultat attendu : `LISTEN 0 128 0.0.0.0:2222`.

### 4.4 Se connecter depuis l'hôte

```bash
ssh -p 2222 clecart@192.168.56.10
```

`-p 2222` est obligatoire, le client SSH visant le port 22 par défaut.

Test que root est bien refusé (à montrer à l'audit) :

```bash
ssh -p 2222 root@192.168.56.10
# -> Permission denied (publickey,password).
```

---

## 5. Pare-feu UFW

`ufw` (*Uncomplicated FireWall*) est une surcouche de `nftables`/`iptables`. Le
principe retenu est le **deny by default** : tout ce qui n'est pas explicitement
autorisé est bloqué.

### 5.1 Règles

```bash
sudo apt install -y ufw

sudo ufw default deny incoming     # tout le trafic entrant est refusé
sudo ufw default allow outgoing    # le serveur peut sortir (apt, DNS, NTP)

sudo ufw allow 2222/tcp comment 'SSH administration'
sudo ufw allow 80/tcp   comment 'HTTP - WordPress'
sudo ufw allow 21/tcp   comment 'FTP - canal de controle'
sudo ufw allow 40000:40100/tcp comment 'FTP - mode passif'
```

> ⚠️ **Autoriser le 2222 AVANT d'activer ufw**, sous peine de se couper soi-même
> la session SSH en cours.

```bash
sudo ufw enable            # répondre 'y'
sudo ufw status verbose
sudo ufw status numbered   # numérote les règles, pour pouvoir en supprimer une
```

`comment '...'` documente chaque règle directement dans le pare-feu : à l'audit,
`ufw status verbose` justifie à lui seul chaque port ouvert.

### 5.2 Pourquoi ces ports, et pourquoi pas les autres

Voir le [récapitulatif détaillé](#13-récapitulatif-des-ports-ouverts-justification-audit).
En résumé : `2222` (administration), `80` (le site), `21` + `40000-40100` (FTP).
Le port `3306` de MySQL n'est **pas** ouvert : la base n'est utilisée que par
WordPress, qui tourne sur la même machine et passe par `127.0.0.1`. Le port `22`
est fermé puisque SSH a déménagé.

### 5.3 Vérifier ce qui écoute réellement

```bash
sudo ss -tulnp
```

`-u` ajoute UDP aux résultats. Chaque ligne doit correspondre soit à un service
volontairement exposé, soit à un service en écoute sur `127.0.0.1` uniquement
(MySQL) — donc inatteignable depuis l'extérieur, quel que soit l'état du
pare-feu. C'est la **défense en profondeur** : deux barrières indépendantes.

---

## 6. Utilisateurs : luffy et zoro

### 6.1 Générer la paire de clés SSH (sur la machine hôte)

**Sur l'hôte**, pas sur le serveur — la clé privée ne doit jamais quitter le
poste de l'administrateur :

```bash
ssh-keygen -t ed25519 -C "clecart@deep-in-system" -f ~/.ssh/deep_in_system
```

| Option | Rôle |
|---|---|
| `-t ed25519` | Algorithme de signature. Ed25519 (courbes elliptiques) est plus court, plus rapide et plus sûr que RSA 2048. Alternative si un vieux client l'exige : `-t rsa -b 4096` |
| `-C "..."` | Commentaire ajouté en fin de clé publique, pour identifier son propriétaire |
| `-f chemin` | Fichier de destination : évite d'écraser une clé existante |

Deux fichiers sont créés :

- `~/.ssh/deep_in_system` → **clé privée**, secrète, jamais transmise (mode `600`)
- `~/.ssh/deep_in_system.pub` → **clé publique**, à copier sur le serveur

La passphrase demandée chiffre la clé privée sur le disque : si le poste est
volé, la clé reste inutilisable. Fortement recommandée.

**Principe de l'authentification par clé** : le serveur envoie un défi aléatoire,
le client le signe avec la clé privée, le serveur vérifie la signature avec la
clé publique. Le secret ne transite **jamais** sur le réseau — contrairement à
un mot de passe, qui est transmis (dans le tunnel chiffré, certes) et peut être
deviné par force brute.

Afficher la clé publique à copier :

```bash
cat ~/.ssh/deep_in_system.pub
# ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... clecart@deep-in-system
```

### 6.2 Créer luffy (sudoer, authentification par clé)

Sur le serveur :

```bash
sudo adduser luffy
```

`adduser` est le script haut niveau de Debian/Ubuntu (à préférer à `useradd`,
bas niveau) : il crée le compte, le groupe personnel, le répertoire
`/home/luffy`, y copie les fichiers de `/etc/skel` (`.bashrc`, `.profile`) et
demande le mot de passe interactivement.

```bash
sudo usermod -aG sudo luffy
```

- `usermod` modifie un compte existant.
- `-a` = *append* : **ajoute** aux groupes, sans retirer les autres.
  Oublier le `-a` avec `-G` remplace tous les groupes secondaires — erreur
  classique qui coupe l'accès de l'utilisateur.
- `-G sudo` : sur Ubuntu, l'appartenance au groupe `sudo` donne le droit
  d'utiliser `sudo` (défini dans `/etc/sudoers` par la ligne
  `%sudo ALL=(ALL:ALL) ALL`).

Vérification :

```bash
id luffy
groups luffy             # -> luffy : luffy sudo
getent passwd luffy      # -> luffy:x:1001:1001::/home/luffy:/bin/bash
```

> ⚠️ Selon la configuration de `/etc/adduser.conf`, `adduser` peut ajouter le
> compte à un groupe supplémentaire `users`. La grille d'audit attend exactement
> `luffy : luffy sudo` et `zoro : zoro` : si `groups` affiche un groupe en trop,
> le retirer avec `sudo gpasswd -d luffy users`.

`getent passwd` interroge la base des comptes (fichier `/etc/passwd` + éventuels
annuaires) : c'est plus fiable qu'un `grep` sur le fichier.

Déposer la clé publique :

```bash
sudo mkdir -p /home/luffy/.ssh
sudo nano /home/luffy/.ssh/authorized_keys
# y coller la ligne complète du fichier .pub, sur UNE seule ligne

sudo chown -R luffy:luffy /home/luffy/.ssh
sudo chmod 700 /home/luffy/.ssh
sudo chmod 600 /home/luffy/.ssh/authorized_keys
```

**Les permissions ne sont pas cosmétiques** : `sshd` refuse purement et
simplement d'utiliser un `authorized_keys` accessible en écriture par le groupe
ou par les autres. Sinon, n'importe quel utilisateur pouvant écrire dans ce
fichier pourrait y ajouter sa propre clé et devenir `luffy`.

| Mode | Signification |
|---|---|
| `700` sur `.ssh/` | `rwx------` : seul le propriétaire entre dans le dossier |
| `600` sur `authorized_keys` | `rw-------` : seul le propriétaire lit et écrit |

Rappel du calcul octal : `4` = lecture (r), `2` = écriture (w), `1` = exécution
(x). Les trois chiffres correspondent à **propriétaire / groupe / autres**.
`700` = 4+2+1 pour le propriétaire, rien pour les autres.

Test depuis l'hôte :

```bash
ssh -p 2222 -i ~/.ssh/deep_in_system luffy@192.168.56.10
```

`-i` désigne l'identité (la clé privée) à présenter. Une fois connecté :

```bash
sudo whoami       # -> root : luffy est bien sudoer
```

### 6.3 Créer zoro (mot de passe, non sudoer)

```bash
sudo adduser zoro
# saisir le mot de passe personnalisé (conservé pour l'audit)
```

Aucune commande supplémentaire : ne **pas** l'ajouter au groupe `sudo`.

Vérifications :

```bash
groups zoro                        # -> zoro : pas de groupe "sudo"
getent passwd zoro                 # -> home = /home/zoro
sudo -l -U zoro                    # -> "User zoro is not allowed to run sudo"
ls -ld /home/zoro
```

Test depuis l'hôte (authentification par mot de passe) :

```bash
ssh -p 2222 zoro@192.168.56.10
```

### 6.4 Politique de mots de passe (bonne pratique)

```bash
sudo apt install -y libpam-pwquality
sudo nano /etc/security/pwquality.conf
```

```
minlen = 10
dcredit = -1      # au moins 1 chiffre
ucredit = -1      # au moins 1 majuscule
lcredit = -1      # au moins 1 minuscule
retry = 3
```

PAM (*Pluggable Authentication Modules*) est la couche d'authentification
commune à tous les services Linux (login console, `sudo`, SSH, vsftpd…). Le
module `pam_pwquality` refuse les mots de passe trop faibles **au moment où ils
sont définis**.

---

## 7. Serveur FTP : vsftpd et l'utilisateur nami

`vsftpd` = *Very Secure FTP Daemon*, le serveur FTP de référence sous Linux,
écrit avec une architecture à séparation de privilèges.

**Objectif du sujet** : `nami` accède en **lecture seule** à `/backup`, et
**uniquement** à `/backup`. Pas d'accès anonyme.

### 7.1 Installation

```bash
sudo apt install -y vsftpd
sudo cp /etc/vsftpd.conf /root/config-backup/vsftpd.conf.orig
```

### 7.2 Créer l'utilisateur nami

```bash
sudo adduser --home /backup --no-create-home --shell /usr/sbin/nologin --gecos "" nami
sudo passwd nami        # définir le mot de passe personnalisé
```

| Option | Rôle |
|---|---|
| `--home /backup` | Le répertoire personnel **est** `/backup` : combiné au *chroot*, `nami` verra `/backup` comme la racine `/` et ne pourra pas en sortir |
| `--no-create-home` | `/backup` existe déjà (c'est une partition) : on ne veut ni le recréer ni y copier `/etc/skel` |
| `--shell /usr/sbin/nologin` | **Aucun shell** : même si le mot de passe fuite, `nami` ne peut pas ouvrir de session SSH ni exécuter de commande. Le compte ne sert qu'au FTP |
| `--gecos ""` | Évite les questions sur le nom complet, le téléphone, etc. |

Pour que PAM accepte cette connexion FTP, le shell doit figurer dans la liste
des shells valides — le module `pam_shells` utilisé par vsftpd la consulte :

```bash
grep -qxF '/usr/sbin/nologin' /etc/shells || echo '/usr/sbin/nologin' | sudo tee -a /etc/shells
```

(`grep -q` silencieux, `-x` ligne entière, `-F` chaîne littérale ; `||` n'exécute
la suite que si le `grep` a échoué, donc si la ligne est absente. `tee -a`
ajoute en fin de fichier au lieu d'écraser.)

### 7.3 Les permissions de /backup : le cœur du « lecture seule »

```bash
sudo chown root:nami /backup
sudo chmod 750 /backup
ls -ld /backup          # -> drwxr-x--- root nami
```

Analyse de `750` = `rwxr-x---` :

| Cible | Droits | Conséquence |
|---|---|---|
| Propriétaire `root` | `rwx` | Le script de sauvegarde (lancé par root) écrit les archives |
| Groupe `nami` | `r-x` | `nami` **liste et traverse** le dossier, mais ne peut **rien y créer ni supprimer** |
| Autres | `---` | Aucun autre utilisateur du système ne voit le contenu des sauvegardes |

C'est la **première** des deux barrières : même si la configuration de vsftpd
était mal faite, le système de fichiers refuserait l'écriture. La seconde
barrière est `write_enable=NO` dans vsftpd (ci-dessous).

Ce mode `750` a un second effet, indispensable : `vsftpd` **refuse de démarrer
une session chrootée si le répertoire racine du chroot est accessible en
écriture par l'utilisateur** (option `allow_writeable_chroot`). Une racine
inscriptible permettrait certaines évasions de chroot. Ici `/backup`
appartient à root et `nami` n'y a pas le droit d'écriture : la contrainte est
naturellement satisfaite.

### 7.4 Configuration de vsftpd

```bash
sudo nano /etc/vsftpd.conf
```

Configuration retenue :

```ini
# --- Fonctionnement général ---
listen=YES
listen_ipv6=NO
pam_service_name=vsftpd
secure_chroot_dir=/var/run/vsftpd/empty
use_localtime=YES

# --- Accès : locaux oui, anonyme non ---
anonymous_enable=NO
local_enable=YES

# --- LECTURE SEULE ---
write_enable=NO
local_umask=022

# --- Enfermer l'utilisateur dans son répertoire personnel ---
chroot_local_user=YES
allow_writeable_chroot=NO

# --- Liste blanche : seul nami a le droit de se connecter ---
userlist_enable=YES
userlist_file=/etc/vsftpd.userlist
userlist_deny=NO

# --- Mode passif ---
pasv_enable=YES
pasv_min_port=40000
pasv_max_port=40100

# --- Journalisation ---
xferlog_enable=YES
xferlog_file=/var/log/vsftpd.log
dirmessage_enable=YES
connect_from_port_20=YES
```

Directives clés :

| Directive | Rôle |
|---|---|
| `anonymous_enable=NO` | **Exigence du sujet.** L'accès anonyme laisserait n'importe qui sur le réseau lire — voire déposer — des fichiers sans identification. Faille majeure |
| `write_enable=NO` | **Exigence du sujet.** Désactive globalement toutes les commandes d'écriture du protocole : `STOR` (envoi), `DELE` (suppression), `RMD`/`MKD` (dossiers), `RNFR`/`RNTO` (renommage) |
| `chroot_local_user=YES` | *change root* : le processus voit le home de l'utilisateur comme `/`. `nami` ne peut donc pas remonter vers `/etc`, `/home` ou `/var` |
| `userlist_enable=YES` + `userlist_deny=NO` | Inverse le sens de la liste : elle devient une **liste blanche**. Seuls les comptes listés peuvent se connecter — `clecart`, `luffy` et `zoro` sont donc refusés en FTP |
| `pasv_min_port` / `pasv_max_port` | En mode **passif**, c'est le client qui ouvre la connexion de données, sur un port annoncé par le serveur. Sans bornes, ce port serait aléatoire dans toute la plage haute et impossible à autoriser proprement dans le pare-feu. On le restreint à 101 ports, ceux ouverts dans ufw |
| `local_umask=022` | Masque de création : fichiers en `644`, dossiers en `755`. Sans effet réel ici puisque l'écriture est interdite, mais cohérent |
| `xferlog_enable=YES` | Journalise chaque transfert : traçabilité |

**Actif vs passif** : en mode *actif*, le serveur initie la connexion de données
vers le client — ce qui est systématiquement bloqué par les pare-feu et les NAT
côté client. Le mode *passif* inverse le sens : c'est le client qui se connecte,
sur un port de la plage définie. C'est le mode utilisé par tous les clients
modernes, d'où l'ouverture de `40000-40100` dans ufw.

Créer la liste blanche :

```bash
echo "nami" | sudo tee /etc/vsftpd.userlist
sudo chmod 600 /etc/vsftpd.userlist
```

Redémarrer et vérifier :

```bash
sudo systemctl restart vsftpd
sudo systemctl status vsftpd
sudo ss -tlnp | grep vsftpd        # -> LISTEN 0.0.0.0:21
```

### 7.5 Tests (depuis la machine hôte)

```bash
ftp 192.168.56.10
# Name: nami
# Password: ********
```

Une fois connecté :

```
ftp> pwd            # -> "/" : la racine vue par nami est en réalité /backup
ftp> ls             # les archives de sauvegarde sont listées
ftp> get wordpress-db-2026-08-25_00-00-01.tar.gz     # doit réussir
ftp> put /etc/hosts test.txt                         # doit ÉCHOUER : 550 Permission denied
ftp> cd ..          # reste bloqué à la racine : le chroot fonctionne
ftp> bye
```

Tests de refus (à montrer à l'audit) :

```bash
ftp 192.168.56.10        # avec le compte "anonymous" -> 530 login incorrect
ftp 192.168.56.10        # avec le compte "clecart"   -> 530 : absent de la liste blanche
```

Si le client hôte n'a pas `ftp` : `sudo apt install ftp` ou utiliser `lftp`,
FileZilla, ou `curl ftp://nami@192.168.56.10/`.

---

## 8. Base de données MySQL

### 8.1 Installation

```bash
sudo apt install -y mysql-server
sudo systemctl status mysql
mysql --version
```

Ubuntu installe MySQL Server 8, déjà démarré et activé au boot.

### 8.2 Sécurisation initiale

```bash
sudo mysql_secure_installation
```

Réponses recommandées :

| Question | Réponse | Pourquoi |
|---|---|---|
| Setup VALIDATE PASSWORD component ? | `y`, niveau `MEDIUM` | Refuse les mots de passe faibles pour les comptes SQL |
| Change the password for root ? | **`n`** | Voir explication ci-dessous |
| Remove anonymous users ? | `y` | Les comptes anonymes permettent de se connecter sans identifiant |
| Disallow root login remotely ? | `y` | **Exigence du sujet** |
| Remove test database ? | `y` | Base `test` accessible à tous par défaut : surface d'attaque inutile |
| Reload privilege tables ? | `y` | Applique immédiatement les changements |

**Pourquoi ne pas définir de mot de passe root MySQL ?**
Sur Ubuntu, `root@localhost` utilise le plugin d'authentification **`auth_socket`** :
MySQL vérifie, via la socket Unix, que l'utilisateur **système** qui se connecte
s'appelle bien `root`. Conséquences :

- il n'existe **aucun mot de passe root à voler ou à forcer** ;
- la connexion n'est possible **que localement**, par la socket Unix
  `/var/run/mysqld/mysqld.sock` — jamais par le réseau, quelle que soit la
  configuration ;
- seul un utilisateur capable de faire `sudo` peut devenir root MySQL, ce qui
  ramène la sécurité de la base à celle du système.

C'est plus robuste qu'un mot de passe, et cela répond directement à
« désactiver la connexion distante à l'utilisateur root ».

### 8.3 Interdire toute connexion venue de l'extérieur

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Vérifier / forcer dans la section `[mysqld]` :

```ini
bind-address = 127.0.0.1
mysqlx-bind-address = 127.0.0.1
```

`bind-address` indique sur quelle **interface** le démon écoute. Avec
`127.0.0.1` (l'interface de bouclage `lo`), le socket TCP n'est joignable que
depuis la machine elle-même : un paquet venu du réseau n'atteint jamais MySQL,
même si le pare-feu était désactivé. `mysqlx-bind-address` fait de même pour le
protocole X (port 33060).

C'est suffisant pour le sujet : WordPress tourne **sur le même serveur** et se
connecte à `localhost`. Aucune fonctionnalité n'est perdue.

```bash
sudo systemctl restart mysql
sudo ss -tlnp | grep 3306
# -> 127.0.0.1:3306   et surtout PAS 0.0.0.0:3306
```

### 8.4 Créer la base et l'utilisateur WordPress

```bash
sudo mysql
```

(Aucun mot de passe demandé : c'est `auth_socket` qui opère.)

```sql
CREATE DATABASE wordpress
  DEFAULT CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'MotDePasseFort_A_Changer';

GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, ALTER, INDEX, REFERENCES
  ON wordpress.* TO 'wp_user'@'localhost';

FLUSH PRIVILEGES;
```

Analyse :

| Élément | Explication |
|---|---|
| `utf8mb4` | Vrai UTF-8 sur 4 octets : gère les caractères accentués **et** les emoji. L'ancien `utf8` de MySQL (3 octets) provoque des erreurs sur certains caractères |
| `utf8mb4_unicode_ci` | Règle de comparaison, insensible à la casse (`ci` = *case insensitive*) |
| `'wp_user'@'localhost'` | En MySQL, un compte est le **couple** identifiant + hôte d'origine. `@'localhost'` signifie : connexion locale uniquement. Un `@'%'` (n'importe quel hôte) serait une faute |
| `ON wordpress.*` | Les privilèges ne portent **que** sur cette base. `wp_user` ne peut ni lire `mysql.user`, ni toucher à une autre base |
| La liste des privilèges | **Exigence du sujet** : « les seuls accès nécessaires ». On évite `ALL PRIVILEGES`, qui inclut `GRANT`, `FILE` (lecture de fichiers du serveur), `SHUTDOWN`, `PROCESS`… `CREATE`, `DROP` et `ALTER` restent nécessaires : WordPress crée ses tables à l'installation et les modifie lors des mises à jour et de l'ajout d'extensions |
| `FLUSH PRIVILEGES` | Recharge les tables de droits en mémoire |

Vérifications :

```sql
SHOW DATABASES;
SELECT user, host, plugin FROM mysql.user;
SHOW GRANTS FOR 'wp_user'@'localhost';
EXIT;
```

`SELECT user, host FROM mysql.user` est **la** commande à montrer à l'audit :
elle prouve qu'aucun compte `root` n'existe avec un hôte autre que `localhost`,
et que `root` utilise bien `auth_socket`.

S'il subsistait un `root` distant, on le supprime :

```sql
DROP USER 'root'@'%';
```

Test de l'utilisateur WordPress :

```bash
mysql -u wp_user -p wordpress -e "SELECT DATABASE();"
```

Et preuve que l'accès distant est impossible, **depuis l'hôte** :

```bash
mysql -h 192.168.56.10 -u wp_user -p    # -> Can't connect / connection refused
```

---

## 9. WordPress

### 9.1 Installer Apache et PHP

```bash
sudo apt install -y apache2 php libapache2-mod-php php-mysql \
  php-curl php-gd php-xml php-mbstring php-zip php-intl php-imagick
```

| Paquet | Rôle |
|---|---|
| `apache2` | Serveur HTTP |
| `php` + `libapache2-mod-php` | Interpréteur PHP intégré à Apache (mod_php) |
| `php-mysql` | Pilote de connexion à MySQL — sans lui, WordPress ne démarre pas |
| `php-gd`, `php-imagick` | Traitement d'images (miniatures) |
| `php-curl`, `php-xml`, `php-zip` | Requêtes HTTP sortantes, flux RSS, installation d'extensions |
| `php-mbstring` | Chaînes multi-octets : indispensable pour l'UTF-8 |

```bash
sudo systemctl status apache2
sudo systemctl enable apache2
```

Test : `http://192.168.56.10/` doit afficher la page « Apache2 Default Page ».

### 9.2 Déployer WordPress à la racine du site

Le sujet impose `http://{host}/` — donc les fichiers vont **directement** dans
la racine documentaire d'Apache (`/var/www/html`), pas dans un sous-dossier
`/wordpress`.

```bash
cd /tmp
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
```

- `wget` télécharge un fichier en HTTP(S).
- `tar -xzf` : `x` extraire, `z` décompresser gzip, `f` depuis ce fichier.
  L'archive contient un dossier `wordpress/`.

```bash
sudo rm -f /var/www/html/index.html          # retire la page par défaut d'Apache
sudo cp -a /tmp/wordpress/. /var/www/html/
```

`cp -a` = archive : copie récursive en préservant les permissions et les liens.
Le `.` final signifie « le **contenu** de `wordpress/` », pas le dossier
lui-même — c'est ce qui place `index.php` à la racine.

### 9.3 Permissions des fichiers

```bash
sudo chown -R www-data:www-data /var/www/html
sudo find /var/www/html -type d -exec chmod 755 {} \;
sudo find /var/www/html -type f -exec chmod 644 {} \;
```

- `www-data` est l'utilisateur système sous lequel tourne Apache. Il doit
  posséder les fichiers pour que WordPress puisse écrire dans `wp-content`
  (uploads, extensions, thèmes).
- `find ... -type d -exec chmod 755 {} \;` applique `755` à tous les
  **répertoires** (`rwxr-xr-x` : le bit `x` est nécessaire pour *traverser* un
  dossier), et `644` à tous les **fichiers** (`rw-r--r--` : aucun fichier de
  contenu web n'a besoin d'être exécutable au sens Unix — PHP est interprété par
  Apache, pas exécuté par le noyau).
- `{}` est remplacé par chaque résultat, `\;` termine la commande de `-exec`
  (échappé pour que le shell ne l'interprète pas).

### 9.4 Configurer wp-config.php

```bash
sudo -u www-data cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
sudo nano /var/www/html/wp-config.php
```

```php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wp_user' );
define( 'DB_PASSWORD', 'MotDePasseFort_A_Changer' );
define( 'DB_HOST', 'localhost' );
define( 'DB_CHARSET', 'utf8mb4' );
```

Remplacer ensuite le bloc des clés de sécurité (`AUTH_KEY`, `SECURE_AUTH_KEY`,
`LOGGED_IN_KEY`, `NONCE_KEY`, et leurs `*_SALT`) par des valeurs générées :

```bash
curl -s https://api.wordpress.org/secret-key/1.1/salt/
```

Ces clés servent à signer et chiffrer les cookies de session. Laisser les
valeurs d'exemple (`put your unique phrase here`) rendrait les cookies
forgeables par n'importe qui — c'est une faille critique, et une vérification
classique en audit.

Durcissement supplémentaire, à ajouter en fin de fichier **avant** la ligne
`/* That's all, stop editing! */` :

```php
define( 'DISALLOW_FILE_EDIT', true );   // désactive l'éditeur de code de l'admin
define( 'WP_DEBUG', false );            // aucun message d'erreur affiché en public
```

`DISALLOW_FILE_EDIT` supprime l'éditeur de thèmes/extensions du tableau de bord :
sans lui, un compte administrateur compromis permet d'écrire du PHP arbitraire
sur le serveur en trois clics.

### 9.5 Rendre wp-config.php inaccessible publiquement

**Exigence du sujet** : `http://{host}/wp-config.php` ne doit rien révéler.

Deux protections complémentaires.

**1) Au niveau du système de fichiers :**

```bash
sudo chown root:www-data /var/www/html/wp-config.php
sudo chmod 640 /var/www/html/wp-config.php
ls -l /var/www/html/wp-config.php     # -> -rw-r----- root www-data
```

Apache (groupe `www-data`) **lit** le fichier ; aucun autre utilisateur du
système ne le peut ; et Apache lui-même ne peut pas le **modifier**.

**2) Au niveau d'Apache :**

```bash
sudo nano /etc/apache2/sites-available/000-default.conf
```

À l'intérieur du bloc `<VirtualHost *:80>` :

```apache
<Directory /var/www/html>
    Options -Indexes +FollowSymLinks
    AllowOverride All
    Require all granted
</Directory>

<Files "wp-config.php">
    Require all denied
</Files>

<FilesMatch "^\.">
    Require all denied
</FilesMatch>

<Files "xmlrpc.php">
    Require all denied
</Files>
```

| Directive | Rôle |
|---|---|
| `Options -Indexes` | Désactive le listage automatique du contenu d'un dossier sans `index.php`. Sinon un visiteur peut parcourir toute l'arborescence du site |
| `AllowOverride All` | Autorise les fichiers `.htaccess`, dont WordPress a besoin pour les permaliens |
| `<Files "wp-config.php"> Require all denied` | Apache renvoie **403 Forbidden** pour ce fichier précis |
| `<FilesMatch "^\.">` | Bloque tous les fichiers cachés (`.env`, `.git`, `.htpasswd`) |
| `<Files "xmlrpc.php">` | `xmlrpc.php` est la cible historique des attaques par amplification et par force brute sur WordPress. Inutile hors application mobile |

> **Pourquoi ne pas se contenter du comportement par défaut ?** Sans règle,
> `http://host/wp-config.php` renvoie une **page blanche** : PHP interprète le
> fichier, qui ne produit aucune sortie. Cela *semble* sûr, mais ne l'est plus
> du tout si le module PHP est désactivé, mal configuré ou tombe en panne — le
> serveur enverrait alors le fichier en clair, avec le mot de passe de la base.
> `Require all denied` protège quel que soit l'état de PHP.

Activer et recharger :

```bash
sudo a2enmod rewrite
sudo apache2ctl configtest        # -> Syntax OK
sudo systemctl reload apache2
```

`a2enmod` (*apache2 enable module*) active `mod_rewrite`, utilisé par les
permaliens. `apache2ctl configtest` valide la syntaxe **avant** de recharger —
même précaution que `sshd -t`.

Durcir la signature du serveur :

```bash
sudo nano /etc/apache2/conf-available/security.conf
```

```apache
ServerTokens Prod
ServerSignature Off
```

Ces deux directives empêchent Apache d'annoncer sa version exacte et celle de
PHP dans ses en-têtes et pages d'erreur — information qui permet à un attaquant
de cibler directement les vulnérabilités connues de cette version.

```bash
sudo systemctl reload apache2
```

### 9.6 Terminer l'installation et tester

Ouvrir `http://192.168.56.10/` depuis l'hôte : l'assistant WordPress apparaît.
Renseigner le titre du site, un identifiant d'administrateur (**pas** `admin`,
identifiant deviné en premier par tous les robots), un mot de passe fort.

Tests obligatoires pour l'audit :

```bash
# la page d'accueil répond
curl -I http://192.168.56.10/
# -> HTTP/1.1 200 OK

# le fichier de configuration est refusé
curl -I http://192.168.56.10/wp-config.php
# -> HTTP/1.1 403 Forbidden

# aucun listage de répertoire
curl -I http://192.168.56.10/wp-content/uploads/
# -> 403 Forbidden
```

Puis, dans le navigateur : publier un article, créer un second utilisateur,
vérifier que l'article s'affiche en page d'accueil. Cela prouve que la chaîne
complète Apache → PHP → MySQL fonctionne.

Vérifier que les données sont bien en base :

```bash
sudo mysql -e "USE wordpress; SHOW TABLES; SELECT post_title FROM wp_posts LIMIT 5;"
```

---

## 10. Sauvegarde automatique par cron

**Exigence du sujet** : tous les jours à 00:00, créer dans `/backup` une archive
`tar` de la base WordPress, dont le **nom contient la date**, et journaliser le
succès et le temps d'exécution dans `/var/log/backup.log`. L'archive doit être
téléchargeable par `nami` en FTP.

### 10.1 Le script de sauvegarde

```bash
sudo nano /usr/local/bin/backup-wordpress.sh
```

```bash
#!/bin/bash
#
# Sauvegarde quotidienne de la base de donnees WordPress.
# Lance par cron (utilisateur root) tous les jours a 00:00.

set -euo pipefail
export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

DB_NAME="wordpress"
BACKUP_DIR="/backup"
LOG_FILE="/var/log/backup.log"
FTP_GROUP="nami"
RETENTION_DAYS=30

START_TS=$(date +%s)
STAMP=$(date +%F_%H-%M-%S)
SQL_FILE="/tmp/${DB_NAME}-${STAMP}.sql"
ARCHIVE="${BACKUP_DIR}/${DB_NAME}-${STAMP}.tar.gz"

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $*" >> "$LOG_FILE"
}

on_error() {
    log "ECHEC - sauvegarde interrompue a la ligne $1"
    rm -f "$SQL_FILE"
    exit 1
}
trap 'on_error $LINENO' ERR

# 1. Export logique de la base
mysqldump --single-transaction --quick --routines --events \
          --databases "$DB_NAME" > "$SQL_FILE"

# 2. Archivage compresse dans /backup
tar -czf "$ARCHIVE" -C /tmp "$(basename "$SQL_FILE")"
rm -f "$SQL_FILE"

# 3. Lisible par nami en FTP, non modifiable
chown root:"$FTP_GROUP" "$ARCHIVE"
chmod 640 "$ARCHIVE"

# 4. Journalisation
DURATION=$(( $(date +%s) - START_TS ))
SIZE=$(du -h "$ARCHIVE" | cut -f1)
log "SUCCESS - wordpress backup created!, date: $(date '+%Y-%m-%d %H:%M:%S'), file: $(basename "$ARCHIVE"), size: ${SIZE}, duration: ${DURATION}s"

# 5. Retention : suppression des archives de plus de 30 jours
find "$BACKUP_DIR" -name "${DB_NAME}-*.tar.gz" -mtime +${RETENTION_DAYS} -delete
```

Rendre le script exécutable et créer le journal :

```bash
sudo chmod 750 /usr/local/bin/backup-wordpress.sh
sudo chown root:root /usr/local/bin/backup-wordpress.sh
sudo touch /var/log/backup.log
sudo chmod 644 /var/log/backup.log
```

`chmod 750` : seul root peut lire et exécuter ce script. Un script lancé par
root et modifiable par un autre utilisateur serait une **élévation de privilèges
immédiate**.

### 10.2 Explication détaillée du script

| Élément | Explication |
|---|---|
| `#!/bin/bash` | *Shebang* : indique au noyau quel interpréteur utiliser |
| `set -e` | Arrête le script dès qu'une commande échoue. Sans cela, un `mysqldump` en erreur produirait une archive vide, silencieusement |
| `set -u` | Erreur si une variable non définie est utilisée (protège des fautes de frappe) |
| `set -o pipefail` | Un pipeline échoue si **n'importe laquelle** de ses commandes échoue, pas seulement la dernière |
| `export PATH=...` | **Indispensable en cron** : cron fournit un environnement minimal, souvent sans `/usr/bin`. Un script qui marche en interactif peut échouer en cron pour cette seule raison |
| `trap '...' ERR` | Installe un gestionnaire d'erreur : toute commande en échec déclenche `on_error`, qui journalise l'échec et nettoie le fichier temporaire. `$LINENO` donne la ligne fautive |
| `date +%F_%H-%M-%S` | Format `2026-08-25_00-00-01`. `%F` = `%Y-%m-%d` (format ISO 8601, donc **triable alphabétiquement**). **Exigence du sujet** : la date est dans le nom du fichier |
| `mysqldump` | Export **logique** : produit un fichier SQL de `CREATE TABLE` et `INSERT` capable de reconstruire la base. Contrairement à une copie brute de `/var/lib/mysql`, il est cohérent, portable et lisible |
| `--single-transaction` | Ouvre une transaction et un instantané cohérent (InnoDB) : la sauvegarde est prise à un instant T précis **sans verrouiller les tables**. Le site reste accessible en écriture pendant le dump |
| `--quick` | Récupère les lignes une par une au lieu de charger toute la table en RAM |
| `--routines --events` | Inclut les procédures stockées et les événements planifiés |
| `--databases "$DB_NAME"` | Ajoute le `CREATE DATABASE` / `USE` dans le dump : la restauration recrée la base même si elle a totalement disparu |
| `tar -czf` | `c` créer, `z` compresser en gzip, `f` vers ce fichier. **Exigence du sujet** : un fichier tar |
| `-C /tmp` | Change de répertoire avant d'archiver : l'archive contient `wordpress-….sql` et non `tmp/wordpress-….sql` |
| `chown root:nami` + `chmod 640` | **Exigence du sujet** : téléchargeable par `nami`. `640` = root lit/écrit, le groupe `nami` **lit seulement**, les autres n'ont rien |
| `DURATION=$(( ... ))` | `$(( ))` est l'évaluation arithmétique du shell ; on soustrait deux horodatages Unix (`date +%s` = secondes depuis 1970) |
| `find -mtime +30 -delete` | Politique de rétention : sans elle, la partition de 6 Go finirait saturée, ce qui ferait échouer **toutes** les sauvegardes suivantes |

Ligne produite dans le journal — elle contient l'**heure d'exécution** et la
**durée**, les deux lectures possibles de « execution time » :

```
2026-08-25 00:00:03 - SUCCESS - wordpress backup created!, date: 2026-08-25 00:00:03, file: wordpress-2026-08-25_00-00-01.tar.gz, size: 412K, duration: 2s
```

Le libellé est calqué sur celui attendu par la grille d'audit
(`<...>wordpress backup created!, date: <...>`) et contient **le succès**,
**l'heure d'exécution** et **la durée**.

> Le journal est en `644` (lisible par tous) et non `640` : l'auditeur le lit
> avec un simple `cat /var/log/backup.log`, **sans `sudo`**. Il ne contient
> aucun secret — uniquement des noms de fichiers et des horodatages.

### 10.3 Tester le script avant de le planifier

```bash
sudo /usr/local/bin/backup-wordpress.sh
ls -lh /backup/
sudo cat /var/log/backup.log
```

Vérifier que l'archive est valide et restaurable :

```bash
tar -tzf /backup/wordpress-*.tar.gz          # -t = lister le contenu sans extraire
```

Procédure de restauration (à savoir expliquer en audit — une sauvegarde jamais
testée n'est pas une sauvegarde) :

```bash
cd /tmp
tar -xzf /backup/wordpress-2026-08-25_00-00-01.tar.gz
sudo mysql < /tmp/wordpress-2026-08-25_00-00-01.sql
```

### 10.4 Planifier la tâche cron

`cron` est le planificateur de tâches d'Unix. Le démon `cron` lit les tables de
tâches (*crontabs*) et exécute chaque commande à l'échéance.

```bash
sudo crontab -e
```

`sudo crontab -e` édite la crontab **de root** — nécessaire ici pour écrire dans
`/backup` et se connecter à MySQL via `auth_socket`. `crontab -e` (sans `sudo`)
éditerait celle de l'utilisateur courant.

Ligne à ajouter :

```cron
0 0 * * * /usr/local/bin/backup-wordpress.sh
```

Format d'une ligne cron — cinq champs de temps, puis la commande :

```
┌─── minute (0-59)
│ ┌─── heure (0-23)
│ │ ┌─── jour du mois (1-31)
│ │ │ ┌─── mois (1-12)
│ │ │ │ ┌─── jour de la semaine (0-7, 0 et 7 = dimanche)
│ │ │ │ │
0 0 * * *  /usr/local/bin/backup-wordpress.sh
```

`*` signifie « toutes les valeurs ». Donc `0 0 * * *` = **minute 0 de l'heure 0,
tous les jours, tous les mois, tous les jours de la semaine** → tous les jours à
minuit pile, ce que demande le sujet.

Vérifications :

```bash
sudo crontab -l                    # affiche la crontab de root
systemctl status cron              # le demon tourne bien
grep CRON /var/log/syslog | tail   # trace des executions passees
timedatectl                        # le fuseau horaire est correct
```

`timedatectl` est important : cron se déclenche sur l'**heure locale** du
serveur. Si le fuseau est en UTC, « 00:00 » ne correspond pas à minuit en heure
française. Pour le régler :

```bash
sudo timedatectl set-timezone Europe/Paris
```

**Test accéléré** (à ne pas oublier de retirer) : remplacer temporairement la
ligne par `* * * * *` pour une exécution chaque minute, attendre deux minutes,
vérifier `/backup` et `/var/log/backup.log`, puis remettre `0 0 * * *`.

### 10.5 Rotation du journal (bonne pratique)

Sans rotation, `/var/log/backup.log` grossit indéfiniment.

```bash
sudo nano /etc/logrotate.d/backup
```

```
/var/log/backup.log {
    monthly
    rotate 12
    compress
    missingok
    notifempty
    create 644 root root
}
```

`logrotate` archive le fichier chaque mois, en conserve 12 versions compressées,
et recrée un fichier vide avec les bons droits.

### 10.6 Vérifier le lien avec le FTP

C'est l'exigence finale du sujet : depuis l'hôte,

```bash
ftp 192.168.56.10       # login nami
ftp> ls                 # les .tar.gz apparaissent
ftp> get wordpress-2026-08-25_00-00-01.tar.gz
ftp> bye
```

---

## 11. Bonus

### 11.1 HTTPS avec un certificat auto-signé

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/clecart-host.key \
  -out /etc/ssl/certs/clecart-host.crt \
  -subj "/C=FR/ST=Normandie/L=Rouen/O=Zone01/CN=clecart-host"

sudo chmod 600 /etc/ssl/private/clecart-host.key
sudo a2enmod ssl
sudo a2ensite default-ssl
```

| Option | Rôle |
|---|---|
| `req -x509` | Génère directement un certificat auto-signé (au lieu d'une demande de signature à envoyer à une autorité) |
| `-nodes` | *No DES* : ne chiffre pas la clé privée par une passphrase, sinon Apache la réclamerait à chaque démarrage |
| `-newkey rsa:2048` | Crée en même temps une clé RSA de 2048 bits |
| `-days 365` | Validité d'un an |
| `-subj` | Renseigne le sujet du certificat sans questions interactives. `CN` = *Common Name*, le nom d'hôte servi |

Éditer `/etc/apache2/sites-available/default-ssl.conf` pour pointer
`SSLCertificateFile` et `SSLCertificateKeyFile` vers ces deux fichiers, puis :

```bash
sudo ufw allow 443/tcp comment 'HTTPS'
sudo apache2ctl configtest && sudo systemctl reload apache2
```

Le navigateur affichera un avertissement : le certificat n'est signé par aucune
autorité reconnue. Le **chiffrement** est pourtant bien réel — ce qui manque est
l'**authentification** de l'identité du serveur.

### 11.2 FTPS (FTP sur TLS)

Dans `/etc/vsftpd.conf` :

```ini
ssl_enable=YES
rsa_cert_file=/etc/ssl/certs/clecart-host.crt
rsa_private_key_file=/etc/ssl/private/clecart-host.key
force_local_logins_ssl=YES
force_local_data_ssl=YES
ssl_tlsv1_2=YES
ssl_sslv2=NO
ssl_sslv3=NO
require_ssl_reuse=NO
```

Sans TLS, **FTP transmet l'identifiant et le mot de passe en clair** sur le
réseau. `force_local_logins_ssl` rend le chiffrement obligatoire pour
l'authentification, `force_local_data_ssl` pour le transfert des fichiers.

```bash
sudo systemctl restart vsftpd
```

Se connecter ensuite avec un client compatible (FileZilla en « FTP explicite sur
TLS », ou `lftp -u nami -e "set ftp:ssl-force true" 192.168.56.10`).

### 11.3 fail2ban

```bash
sudo apt install -y fail2ban
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5

[sshd]
enabled = true
port    = 2222

[vsftpd]
enabled = true
```

`fail2ban` surveille les journaux d'authentification et bannit dynamiquement
(via le pare-feu) toute IP qui échoue 5 fois en 10 minutes. C'est le complément
naturel de `MaxAuthTries` : la contre-mesure passe du niveau applicatif au
niveau réseau.

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

### 11.4 Idées supplémentaires

- **Serveur Minecraft** encapsulé dans une unité `systemd` (`Restart=always`,
  `WantedBy=multi-user.target`) pour redémarrer automatiquement après un reboot.
- **Ansible** : rejouer l'intégralité de cette configuration sous forme de rôles
  (`network`, `ssh`, `firewall`, `users`, `ftp`, `mysql`, `wordpress`, `backup`)
  afin de reconstruire un serveur identique en une commande.
- **`unattended-upgrades`** pour l'application automatique des correctifs de
  sécurité.
- **`aide`** ou **`auditd`** pour la détection de modification de fichiers
  système.

---

## 12. Export OVA, sha1 et rendu

### 12.0 Contrôles exigés par la grille d'audit

**La distribution doit être un Ubuntu Server LTS, pas un Desktop :**

```bash
cat /etc/os-release          # -> PRETTY_NAME="Ubuntu XX.04.Y LTS"
dpkg -l ubuntu-desktop       # -> "no packages found matching ubuntu-desktop"
```

`dpkg -l` interroge la base des paquets installés. L'absence du méta-paquet
`ubuntu-desktop` prouve qu'aucun environnement graphique n'a été installé — un
serveur n'en a pas besoin, et chaque paquet superflu est une surface d'attaque
et une consommation de ressources en plus.

**La VM ne doit contenir aucun alias susceptible de fausser les commandes
d'audit :**

```bash
alias                                   # aucun alias personnalise
type -a ufw sudo ip hostname cat ls     # doivent pointer vers /usr/bin, /usr/sbin...
grep -rn "alias" ~/.bashrc ~/.bash_aliases /etc/profile.d/ 2>/dev/null
```

L'auditeur vérifie ce point car un alias comme `alias ufw='echo "Status: active"'`
permettrait de simuler une configuration inexistante. Les alias par défaut
d'Ubuntu (`ll`, `la`, `l`, `alert`) présents dans `~/.bashrc` sont sans effet sur
les commandes vérifiées et peuvent rester. **Ne rien ajouter d'autre**, et
vérifier aussi pour `luffy`, `zoro` et `root`.

```bash
sudo -u luffy bash -ic alias
sudo bash -ic alias
```

### 12.1 Préparer la VM

```bash
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y
sudo apt clean                       # vide le cache des .deb telecharges
history -c                           # nettoie l'historique du shell
sudo poweroff
```

La VM doit être **éteinte proprement** avant l'export : un export à chaud
produit une image dont le système de fichiers n'est pas cohérent.

### 12.2 Exporter

En ligne de commande sur l'hôte :

```bash
VBoxManage export deep-in-system -o ~/deep-in-system.ova
```

Ou par l'interface : **Fichier → Exporter un appareil virtuel → OVF 2.0**.

Le format **OVA** (*Open Virtualization Appliance*) est une archive `tar`
contenant le descripteur OVF (matériel virtuel, réseau) et le disque au format
VMDK. C'est un format ouvert, réimportable dans VirtualBox comme dans VMware.

### 12.3 Calculer et publier l'empreinte

```bash
cd ~
sync                                              # <- indispensable, voir ci-dessous
sha1sum deep-in-system.ova > deep-in-system.sha1
cp deep-in-system.sha1 DeepInSystem.sha1
cat deep-in-system.sha1 | cat -e
```

> ⚠️ **Le `sync` n'est pas décoratif.** `VBoxManage export` rend la main dès que
> l'écriture est *demandée*, pas terminée : sur un fichier de plusieurs Go, une
> partie reste dans le cache d'écriture du noyau. Une empreinte calculée
> immédiatement porte alors sur un fichier incomplet — et ne correspondra plus
> une fois les données réellement écrites.
>
> Cette erreur fait échouer **la toute première question de l'audit** (« *Is the
> SHA1 of the provided machine the same as the machine being audited?* »).
> `sync` force le vidage des caches. Vérifier ensuite la stabilité :
>
> ```bash
> sha1sum deep-in-system.ova ; sha1sum deep-in-system.ova
> ```
>
> Les deux lignes doivent être identiques. On peut aussi recouper avec un autre
> outil : `openssl dgst -sha1 deep-in-system.ova`.

> ⚠️ **Le sujet et la grille d'audit ne donnent pas le même nom de fichier.**
> Le sujet demande `deep-in-system.sha1`, la grille d'audit vérifie la présence
> de `DeepInSystem.sha1`. On dépose donc **les deux fichiers**, au contenu
> identique : aucun des deux critères ne peut être manqué.

> ⚠️ **Nommer l'OVA exactement `deep-in-system.ova`.** L'auditeur exécute
> `sha1sum {ova} > deep-in-system-toaudit.sha1` puis `diff` avec ton fichier. Or
> `sha1sum` écrit `<empreinte>  <nom du fichier>` : si le fichier a été renommé
> entre-temps, le `diff` échoue **alors que l'empreinte est identique**. En cas
> de litige, comparer uniquement les empreintes :
> `awk '{print $1}' deep-in-system.sha1` de chaque côté.

- `sha1sum` calcule une empreinte cryptographique du fichier : toute
  modification, même d'un seul bit, change complètement le résultat. C'est ce
  qui garantit à l'auditeur que la VM auditée est **exactement** celle qui a été
  rendue.
- `cat -e` affiche les caractères de fin de ligne (`$`) : cela permet de
  vérifier qu'il n'y a ni retour chariot Windows (`^M`) ni ligne parasite.

> ⚠️ **Ne plus jamais démarrer ni ré-exporter la VM** après ce calcul : la
> moindre écriture disque change l'empreinte et le rendu ne correspondra plus.

### 12.4 Vérifier l'OVA — l'étape que tout le monde saute

**`Successfully exported` ne garantit rien.** Lors de cette installation, un
premier export s'est terminé sur ce message alors que le disque virtuel qu'il
contenait était corrompu. L'erreur n'apparaît qu'à l'import :

```
VBoxManage: error: Appliance import failed
VBoxManage: error: VMDK: Compressed image is corrupted (VERR_ZIP_CORRUPTED)
```

Le jour de l'audit, l'examinateur n'aurait pas pu démarrer la machine — échec
avant même la première question. **Le seul contrôle qui vaille est de réimporter
l'OVA et de le faire tourner**, exactement comme le fera l'auditeur.

```bash
# 1. Reimporter sous un nom distinct pour ne pas ecraser la VM d'origine
VBoxManage import ~/deep-in-system.ova --vsys 0 --vmname deep-in-system-VERIF

# 2. La demarrer sans interface graphique
VBoxManage startvm deep-in-system-VERIF --type headless

# 3. Derouler les controles dessus (voir §17)
ssh -i ~/.ssh/deep_in_system -p 2222 clecart@192.168.56.10

# 4. Une fois valide, la supprimer avec son disque
VBoxManage controlvm deep-in-system-VERIF poweroff
VBoxManage unregistervm deep-in-system-VERIF --delete
```

Deux enseignements de ce test :

- **Les adresses MAC changent à l'import.** VirtualBox en régénère de nouvelles.
  La configuration réseau doit donc cibler les **noms d'interfaces**
  (`enp0s3`, `enp0s8`) et non les adresses MAC : un fichier netplan utilisant
  `match: macaddress:` laisserait la machine sans réseau chez l'auditeur.
- **Calculer l'empreinte quand la machine est au repos.** Une lecture faite
  pendant que VirtualBox écrit ou supprime plusieurs Go renvoie des valeurs
  incohérentes. Confirmer avec plusieurs lectures et deux outils :

```bash
sha1sum deep-in-system.ova
sha1sum deep-in-system.ova
openssl dgst -sha1 deep-in-system.ova
```

Les trois doivent concorder. Si ce n'est pas le cas, attendre la fin des
écritures en cours et recommencer.

### 12.5 Pousser sur le dépôt

```bash
cp ~/deep-in-system.sha1 ~/DeepInSystem.sha1 "/home/zone01student/dev/Master Bac +5/deep-in-system/"
cd "/home/zone01student/dev/Master Bac +5/deep-in-system"
git add README.md deep-in-system.sha1 DeepInSystem.sha1
git commit -m "deep-in-system: documentation et empreinte de la VM"
git push origin main
```

Le dépôt doit contenir :

```
deep-in-system/
├── README.md
├── deep-in-system.sha1      <- nom demandé par le sujet
└── DeepInSystem.sha1        <- nom vérifié par la grille d'audit
```

Le fichier `.ova` lui-même n'est **pas** versionné (plusieurs Go) : il est
conservé à part et apporté le jour de l'audit.

---

## 13. Récapitulatif des ports ouverts (justification audit)

```bash
sudo ufw status verbose
```

| Port | Protocole | Service | Justification |
|---|---|---|---|
| **2222** | TCP | OpenSSH | Seul canal d'administration du serveur. Déplacé du 22 pour échapper aux scans automatisés. Root interdit, 3 tentatives maximum |
| **80** | TCP | Apache / WordPress | Le site doit être servi sur `http://{host}/` — c'est la raison d'être du serveur |
| **21** | TCP | vsftpd (contrôle) | Canal de commandes FTP. Nécessaire pour que `nami` récupère les sauvegardes. Accès anonyme désactivé, liste blanche d'un seul compte, lecture seule |
| **40000-40100** | TCP | vsftpd (données passives) | En mode passif, le transfert de fichiers utilise un second port choisi dans cette plage. Sans elle, aucun téléchargement ne peut aboutir. La plage est volontairement étroite (101 ports) |
| **443** | TCP | Apache TLS | *Uniquement si le bonus HTTPS est activé* |

**Ports volontairement fermés :**

| Port | Service | Pourquoi il est fermé |
|---|---|---|
| 22 | SSH par défaut | SSH a été déplacé sur 2222 |
| 3306 | MySQL | La base n'est utilisée que par WordPress, sur la même machine, via `127.0.0.1`. Double protection : `bind-address=127.0.0.1` **et** absence de règle ufw |
| 25 / 110 / 143 | Messagerie | Aucun service de mail sur ce serveur |
| tout le reste | — | Politique `deny incoming` par défaut : ce qui n'est pas listé est refusé |

Le trafic **sortant** est autorisé (`allow outgoing`) : le serveur doit pouvoir
joindre les dépôts `apt` (mises à jour de sécurité), le DNS et NTP.

---

## 14. Mémo audit : créer l'utilisateur kratos en moins de 10 minutes

> **Cette épreuve est éliminatoire.** La grille est explicite : *« If the student
> can't solve this exam, he must directly fail in this project. »* Le compte
> s'appelle obligatoirement **`kratos`**, il doit être **sudoer**, la clé privée
> doit être **générée pendant l'épreuve** (pas réutilisée), et le tout en
> **moins de 10 minutes**. Il faut ensuite démontrer deux choses : la connexion
> par clé **et** l'exécution d'une commande `sudo`.

Séquence à connaître par cœur — à répéter plusieurs fois avant l'audit :

```bash
# 1. Sur la machine cliente : generer la paire de cles
ssh-keygen -t ed25519 -f ~/.ssh/audit_key
cat ~/.ssh/audit_key.pub

# 2. Sur le serveur : creer le compte et le rendre sudoer
sudo adduser kratos
sudo usermod -aG sudo kratos

# 3. Installer la cle publique
sudo mkdir -p /home/kratos/.ssh
sudo nano /home/kratos/.ssh/authorized_keys      # coller la cle publique
sudo chown -R kratos:kratos /home/kratos/.ssh
sudo chmod 700 /home/kratos/.ssh
sudo chmod 600 /home/kratos/.ssh/authorized_keys

# 4. Verifier
id kratos
sudo ssh-keygen -lf /home/kratos/.ssh/authorized_keys

# 5. Tester depuis le client
ssh -p 2222 -i ~/.ssh/audit_key kratos@192.168.56.10
sudo whoami        # -> root
```

**Variante en une seule commande** si le compte a déjà un mot de passe (depuis
le client) :

```bash
ssh-copy-id -i ~/.ssh/audit_key.pub -p 2222 kratos@192.168.56.10
```

`ssh-copy-id` crée `.ssh`, ajoute la clé à `authorized_keys` et pose les
permissions correctes automatiquement.

**Les trois erreurs qui font échouer ce test :**

1. Permissions : `.ssh` doit être en `700`, `authorized_keys` en `600`, et les
   deux doivent **appartenir à l'utilisateur** — pas à root après un `sudo nano`.
2. Clé coupée : `authorized_keys` doit contenir la clé sur **une seule ligne**.
3. Oubli de `-p 2222` côté client.

En cas de refus, le diagnostic se fait toujours des deux côtés :

```bash
ssh -vvv -p 2222 -i ~/.ssh/audit_key kratos@192.168.56.10   # cote client
sudo journalctl -u ssh -f                                   # cote serveur
```

Chronométrage réaliste : environ 30 secondes pour la clé, 1 minute pour le
compte, 1 minute pour la clé publique, 30 secondes pour le test. Les 7 minutes
restantes servent à corriger une éventuelle erreur de permissions.

**Répétition à faire avant l'audit** : dérouler cette séquence en entier sur ta
VM avec un utilisateur jetable, puis supprimer le compte :

```bash
sudo deluser --remove-home testuser
```


---

## 15. Checklist finale avant audit

| # | Vérification | Commande | Bloc d'audit |
|---|---|---|---|
| 1 | `DeepInSystem.sha1` **et** `deep-in-system.sha1` dans le dépôt | `git ls-files` | General |
| 2 | Empreinte de l'OVA identique | `sha1sum deep-in-system.ova` puis `diff` | General |
| 3 | Aucun alias parasite | `alias` ; `type -a ufw sudo ip cat` | General |
| 4 | Ubuntu **Server** LTS, pas Desktop | `cat /etc/os-release` ; `dpkg -l ubuntu-desktop` | VM |
| 5 | Disque 30 G, 4 partitions aux bonnes tailles | `lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINT /dev/sda` | VM |
| 6 | Swap de 4 Go actif | `swapon --show` | VM |
| 7 | Hostname `clecart-host` | `hostname` | VM |
| 8 | Utilisateur `clecart` (≠ root), dans le groupe `sudo` | `id` | VM |
| 9 | Aucune interface en dynamique | `ip a \| grep dynamic` → **vide** | Réseau |
| 10 | Accès Internet | `ping -c 5 google.com` | Réseau |
| 11 | Savoir montrer et expliquer `/etc/netplan/01-static-config.yaml` | — | Réseau |
| 12 | SSH sur 2222, root refusé | `sudo sshd -T \| grep -E 'port\|permitrootlogin'` | Réseau |
| 13 | Connexion SSH **depuis l'extérieur** en mot de passe | `ssh clecart@192.168.56.10 -p 2222` | Réseau |
| 14 | Pare-feu actif, ports justifiés, 3306 fermé | `sudo ufw status verbose` | Réseau |
| 15 | luffy : connexion par clé **sans mot de passe** | `ssh -p 2222 -i clé luffy@…` | Users |
| 16 | luffy : `luffy : luffy sudo` et home `/home/luffy` | `groups luffy` ; `echo ~` ; `echo $HOME` | Users |
| 17 | zoro : connexion par mot de passe | `ssh -p 2222 zoro@…` | Users |
| 18 | zoro : sudo refusé, pas dans le groupe sudo | `sudo cat /etc/shadow` ; `groups zoro` | Users |
| 19 | **Épreuve `kratos` réussie en < 10 min** | voir [§14](#14-mémo-audit--créer-lutilisateur-kratos-en-moins-de-10-minutes) | Users |
| 20 | Fichier créé par l'auditeur visible et téléchargeable en FTP | `sudo touch /backup/audit-check` puis `get` | Services |
| 21 | FTP anonyme refusé (mot de passe vide) | login `anonymous` → `530 Login incorrect` | Services |
| 22 | WordPress fonctionnel, connexion admin, publication | navigateur sur `http://192.168.56.10/` | WordPress |
| 23 | `wp-config.php` non affiché | `curl -I http://…/wp-config.php` → 403 | WordPress |
| 24 | Cron `0 0 * * *` créant un tar de la base dans `/backup` | `sudo crontab -l` | Backup |
| 25 | Test `* * * * *` : archive du jour visible en FTP | vider `/backup`, attendre 1 min | Backup |
| 26 | `/var/log/backup.log` lisible **sans sudo**, succès + horodatage | `cat /var/log/backup.log` | Backup |
| 27 | Savoir répondre aux questions de cours | voir [§16](#16-questions-daudit--réponses-types) | Toutes |
| 28 | Tout survit à un redémarrage | `sudo reboot` puis rejouer 1 à 27 | Toutes |

> Le point **28** est le plus important : la quasi-totalité des échecs d'audit
> vient d'une configuration appliquée à chaud mais jamais persistée (réseau
> réécrit par cloud-init, service non `enable`, règle de pare-feu non
> sauvegardée). **Toujours auditer sa propre VM après un reboot complet.**
>
> Le point **25** est le second piège : l'auditeur **supprime** `/var/log/backup.log`
> avant le test. Le script doit donc savoir recréer ce fichier tout seul — c'est le
> cas ici, la redirection `>>` le recrée, et root a un `umask` de `022` qui lui
> redonne le mode `644` attendu.

---

---

## 16. Questions d'audit — réponses types

La grille contient neuf questions de compréhension notées séparément. Voici les
réponses attendues.

### 16.1 « Qu'est-ce que le groupe sudo sous Linux ? »

`sudo` permet d'exécuter **une commande précise** avec les privilèges d'un autre
utilisateur, root par défaut. Qui a le droit de l'utiliser est défini dans
`/etc/sudoers`, qui contient sur Ubuntu la ligne :

```
%sudo   ALL=(ALL:ALL) ALL
```

Le `%` désigne un **groupe**. Cette ligne signifie donc : « tout membre du groupe
`sudo` peut, depuis n'importe quel hôte, exécuter n'importe quelle commande en
tant que n'importe quel utilisateur ». L'appartenance au groupe `sudo` est donc,
sur Ubuntu, ce qui donne les droits d'administration.

**Pourquoi passer par `sudo` plutôt que par le compte root :**

| | `su -` / connexion root | `sudo` |
|---|---|---|
| Mot de passe | Le mot de passe **de root**, partagé entre tous les admins | Le mot de passe **de l'utilisateur**, personnel |
| Traçabilité | Toutes les actions sont attribuées à « root » | Chaque commande est journalisée avec le nom réel dans `/var/log/auth.log` |
| Portée | **Toute** la session tourne en privilégié, y compris les commandes anodines | Seule la commande demandée est privilégiée |
| Révocation | Il faut changer le mot de passe root et le rediffuser | `gpasswd -d user sudo` : immédiat et individuel |
| Granularité | Tout ou rien | On peut n'autoriser que certaines commandes (`user ALL=(root) /usr/bin/systemctl restart apache2`) |

Le risque évité est concret : dans un shell root permanent, une faute de frappe
dans un `rm` détruit le système. Avec `sudo`, la même faute de frappe échoue sur
un refus de permission.

Éditer `/etc/sudoers` se fait **toujours** avec `visudo`, qui valide la syntaxe
avant d'enregistrer : un fichier `sudoers` invalide bloque `sudo` pour tout le
monde, y compris pour le réparer.

### 16.2 « Expliquez votre configuration réseau »

Fichier à montrer : `/etc/netplan/01-static-config.yaml` (détaillé en
[§3.3](#33-écrire-la-configuration-statique)). Points à énoncer :

1. **netplan** n'est qu'un descripteur en YAML ; il **génère** la configuration
   du véritable gestionnaire réseau, ici `systemd-networkd` (`renderer`).
2. `dhcp4: false`, `dhcp6: false` et `accept-ra: false` suppriment **les trois**
   mécanismes d'attribution automatique d'adresse (DHCP v4, DHCP v6, SLAAC).
3. `addresses` fixe l'adresse et le masque, `routes` la passerelle par défaut,
   `nameservers` les serveurs DNS.
4. Il faut aussi montrer `/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg`,
   sans lequel **cloud-init réécrirait tout au prochain démarrage**.

### 16.3 « Qu'est-ce qu'un masque de sous-réseau (netmask) ? »

Une adresse IPv4 fait 32 bits. Le masque indique **où couper** cette adresse
entre une **partie réseau** (commune à toutes les machines du même lien) et une
**partie hôte** (qui identifie la machine sur ce lien).

Pour `192.168.56.10/24` :

```
IP      192.168.56.10   -> 11000000.10101000.00111000.00001010
Masque  255.255.255.0   -> 11111111.11111111.11111111.00000000
                           \_________ réseau _________/\_ hôte _/
Réseau  192.168.56.0     (partie hôte à 0)
Diffusion 192.168.56.255 (partie hôte à 1)
```

`/24` est la **notation CIDR** : 24 bits à 1 dans le masque. Il reste 8 bits
d'hôte, soit 2⁸ = 256 adresses, dont **254 utilisables** — l'adresse de réseau
(`.0`) et l'adresse de diffusion (`.255`) sont réservées.

**À quoi il sert concrètement** : avant d'émettre un paquet, la machine applique
un ET binaire entre le masque et l'adresse de destination, puis compare le
résultat à son propre réseau.

- Résultats égaux → le destinataire est sur le **même lien** : la machine le
  joint directement (résolution ARP puis envoi sur le réseau local).
- Résultats différents → le destinataire est **ailleurs** : le paquet part vers
  la **passerelle par défaut**, qui se chargera de le router.

Le masque est donc ce qui permet à une machine de décider, pour chaque paquet,
entre « livraison directe » et « passage par le routeur ».

### 16.4 « Pourquoi une adresse IP statique est-elle importante pour un serveur web ? »

Parce qu'un serveur est, par définition, la partie **que les clients doivent
pouvoir retrouver**. Si son adresse change, plus rien ne le joint.

| Élément qui dépend de l'adresse | Ce qui casse si elle change |
|---|---|
| Enregistrement DNS (`A`) | Le nom de domaine pointe vers une adresse qui n'est plus la bonne : site injoignable jusqu'à la mise à jour et la fin du cache DNS |
| Règles de pare-feu et de NAT | Les redirections de ports du routeur visent l'ancienne adresse |
| Certificats TLS, `wp-config.php`, `siteurl` de WordPress | Configurations qui référencent l'hôte |
| Sessions SSH / FTP des administrateurs | Les scripts et les clients pointent l'ancienne adresse |
| Journaux et supervision | L'historique devient incohérent, la corrélation d'incidents impossible |

Avec DHCP, l'adresse est un **bail temporaire** : elle peut changer à
l'expiration du bail, après un redémarrage du serveur ou du routeur, ou si une
autre machine prend la place. C'est acceptable pour un poste client, qui ne fait
qu'**initier** des connexions ; c'est inacceptable pour un serveur, qui doit les
**recevoir**.

Ajoutons que le service DHCP devient un point de défaillance unique : s'il est
indisponible au démarrage, un serveur en DHCP peut se retrouver sans adresse du
tout.

### 16.5 « Qu'est-ce qu'un serveur SSH et quel est son rôle ? »

SSH (*Secure Shell*) est un protocole d'accès distant **chiffré**. Le programme
serveur, `sshd`, écoute sur un port (22 par défaut, **2222** ici) et ouvre, pour
chaque client authentifié, une session shell distante.

Il remplace `telnet` et `rlogin`, qui transmettaient **identifiants, mots de
passe et commandes en clair** : n'importe qui sur le chemin réseau pouvait les
lire.

Les trois garanties apportées :

1. **Confidentialité** — après un échange de clés Diffie-Hellman, tout le trafic
   est chiffré symétriquement. Un observateur ne voit que des octets illisibles.
2. **Intégrité** — chaque paquet porte un code d'authentification (MAC) : toute
   modification en transit est détectée.
3. **Authentification** — dans **les deux sens**. Le client prouve son identité
   (mot de passe ou paire de clés), et le serveur prouve la sienne grâce à sa
   clé d'hôte — c'est l'empreinte que le client mémorise à la première connexion
   et qui protège de l'attaque de l'intercepteur (*man-in-the-middle*).

SSH sert aussi de **transport** à d'autres usages : `scp` et `sftp` (transfert
de fichiers), `rsync` sur SSH, redirections de ports (tunnels), `git`.

Ici, c'est l'unique canal d'administration du serveur, d'où son durcissement :
port déplacé, root interdit, `luffy` restreint à l'authentification par clé.

### 16.6 « Qu'est-ce qu'un pare-feu et quel est son rôle sur un serveur ? »

Un pare-feu filtre les paquets réseau selon des règles portant sur l'adresse
source et destination, le port, le protocole et l'état de la connexion. Sous
Linux, le filtrage est assuré par le noyau (*netfilter*) ; `iptables` et
`nftables` en sont les interfaces, et `ufw` une surcouche simplifiée.

**Son rôle : réduire la surface d'attaque.** Un service peut écouter sur un port
sans qu'on le sache — installé comme dépendance, laissé actif par défaut, ou
ajouté par un attaquant. Le pare-feu garantit que **seuls les ports
explicitement autorisés sont joignables**, indépendamment de ce qui tourne sur
la machine.

D'où la politique retenue, `deny incoming` par défaut : on n'énumère pas ce
qu'on interdit (liste sans fin, jamais à jour), on énumère ce qu'on autorise.
C'est le principe de la **liste blanche**, et celui du **moindre privilège**
appliqué au réseau.

C'est aussi une **défense en profondeur** : MySQL est protégé deux fois, par
`bind-address=127.0.0.1` (il n'écoute pas sur le réseau) **et** par l'absence de
règle ufw pour le port 3306. Si l'une des deux protections tombe — erreur de
configuration, mise à jour qui réinitialise un fichier — l'autre tient encore.

Ce qu'un pare-feu ne fait **pas** : il ne protège pas contre une faille du
service qu'on a volontairement exposé. Le port 80 est ouvert, donc une
vulnérabilité de WordPress reste exploitable. D'où les autres mesures :
mises à jour, `DISALLOW_FILE_EDIT`, utilisateur MySQL à privilèges limités.

### 16.7 « Justifiez chaque port ouvert »

Voir le tableau complet en
[§13](#13-récapitulatif-des-ports-ouverts-justification-audit).

Formulation courte : **2222** (SSH, seul canal d'administration), **80** (le site
WordPress, raison d'être du serveur), **21** (canal de contrôle FTP pour que
`nami` récupère les sauvegardes), **40000-40100** (canal de données FTP en mode
passif). Tout le reste est fermé par la politique par défaut, **et notamment le
3306 de MySQL**, qui n'a aucune raison d'être joignable de l'extérieur puisque
WordPress tourne sur la même machine.

### 16.8 « Qu'est-ce qu'un serveur FTP et quel est son rôle ? »

FTP (*File Transfer Protocol*, RFC 959) est un protocole dédié au **transfert de
fichiers** entre un client et un serveur : lister un répertoire, télécharger,
déposer, renommer, supprimer.

Sa particularité est d'utiliser **deux connexions TCP distinctes** :

- le **canal de contrôle** (port 21) transporte les commandes et les réponses ;
- le **canal de données**, ouvert pour chaque transfert, transporte le contenu
  des fichiers.

C'est cette séparation qui impose la distinction **actif / passif** : en mode
actif, c'est le serveur qui ouvre la connexion de données vers le client — ce
que bloquent les pare-feu côté client ; en mode passif, c'est le client qui se
connecte à un port annoncé par le serveur, d'où la plage `40000-40100` ouverte
dans ufw.

**Son rôle ici** : mettre les archives de sauvegarde à disposition de `nami`,
en lecture seule et sans lui donner le moindre accès au reste du système —
`nami` n'a pas de shell et est enfermé par *chroot* dans `/backup`.

**Sa limite, à savoir dire** : FTP transmet **les identifiants et les données en
clair**. En production on utilise **FTPS** (FTP sur TLS, cf.
[§11.2](#112-ftps-ftp-sur-tls)) ou **SFTP** (transfert dans un tunnel SSH, sans
rapport avec FTP malgré le nom).

### 16.9 « Qu'est-ce qu'une tâche cron et quel est son rôle ? »

`cron` est le **planificateur de tâches** d'Unix. Le démon `cron` se réveille
chaque minute, lit les tables de tâches (*crontabs*) et exécute les commandes
dont l'échéance est atteinte.

Une ligne se compose de cinq champs de temps — minute, heure, jour du mois,
mois, jour de la semaine — suivis de la commande. `0 0 * * *` signifie « à la
minute 0 de l'heure 0, tous les jours » : minuit pile.

**Son rôle** : automatiser les tâches récurrentes — sauvegardes, rotation des
journaux, purge de fichiers temporaires, mises à jour de sécurité, envoi de
rapports. L'intérêt n'est pas seulement le confort : une tâche automatisée
s'exécute **même quand l'administrateur est absent, malade ou l'a oubliée**.
Une sauvegarde qui dépend d'une action humaine quotidienne n'est pas fiable.

Les deux pièges classiques, tous deux traités dans le script :

1. **L'environnement est minimal** : `PATH` est réduit, aucun profil shell n'est
   chargé. Un script qui fonctionne en interactif peut échouer en cron — d'où le
   `export PATH=...` en tête de script et les chemins absolus.
2. **La sortie n'est vue par personne** : sans journalisation explicite dans un
   fichier, un échec passe totalement inaperçu — d'où `/var/log/backup.log` et
   le `trap ERR`.

### 16.10 « Pourquoi les sauvegardes sont-elles importantes ? »

Parce qu'une sauvegarde est **le seul moyen de revenir à un état antérieur
connu**. Aucune autre mesure de sécurité ne le permet.

Les causes de perte de données, par fréquence réelle décroissante :

| Cause | Exemple |
|---|---|
| **Erreur humaine** | `DROP TABLE` sur la mauvaise base, `rm -rf` mal ciblé, mise à jour d'extension qui casse le site |
| **Défaillance logicielle** | Corruption de la base après une coupure, migration ratée |
| **Panne matérielle** | Disque HS, secteurs défectueux |
| **Malveillance** | Rançongiciel, défiguration du site, intrusion |
| **Sinistre** | Incendie, dégât des eaux, panne électrique prolongée du centre de données |

Un pare-feu, un antivirus ou des mots de passe forts ne protègent **d'aucune**
des deux premières lignes, qui sont les plus courantes.

Trois notions à citer :

- **Règle 3-2-1** : 3 copies des données, sur 2 supports différents, dont 1 hors
  site. Ici, la séparation `/` et `/backup` n'est qu'un premier niveau ; le
  téléchargement FTP par `nami` permet la copie hors machine.
- **RPO** (*Recovery Point Objective*) : quantité de données qu'on accepte de
  perdre. Avec une sauvegarde quotidienne à minuit, le RPO est de **24 heures**.
- **RTO** (*Recovery Time Objective*) : temps nécessaire pour être de nouveau
  opérationnel.

Enfin, la phrase qui résume tout : **une sauvegarde jamais restaurée n'est pas
une sauvegarde**, c'est une hypothèse. D'où la procédure de restauration testée
en [§10.3](#103-tester-le-script-avant-de-le-planifier).

---

## 17. Grille d'audit officielle — commandes et sorties attendues

Sorties littérales que l'auditeur va observer. À vérifier une dernière fois
**après un redémarrage complet**.

### Général

```console
$ git ls-files
DeepInSystem.sha1
README.md
deep-in-system.sha1

$ alias                       # rien d'autre que les alias par defaut d'Ubuntu
$ dpkg -l ubuntu-desktop
dpkg-query: no packages found matching ubuntu-desktop
```

### Machine virtuelle

```console
$ cat /etc/os-release | head -2
PRETTY_NAME="Ubuntu 24.04.3 LTS"
NAME="Ubuntu"

$ lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINT /dev/sda
NAME   FSTYPE   SIZE MOUNTPOINT
sda               30G
├─sda1             1M
├─sda2 swap        4G [SWAP]
├─sda3 ext4       15G /
├─sda4 ext4        5G /home
└─sda5 ext4        6G /backup

$ hostname
clecart-host

$ id
uid=1000(clecart) gid=1000(clecart) groups=1000(clecart),27(sudo),...
```

> La ligne `1M` est la partition `bios_grub` — elle apparaît **aussi dans
> l'exemple de la grille d'audit**, ce qui confirme que le démarrage en BIOS est
> bien la configuration attendue. Tolérance annoncée sur les tailles : ±0,5 G.

### Réseau et sécurité

```console
$ ip a | grep dynamic
$                             # <- AUCUNE sortie

$ ping -c 5 google.com
5 packets transmitted, 5 received, 0% packet loss

$ sudo sshd -T | grep -E "^(port|permitrootlogin)"
port 2222
permitrootlogin no

$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ------
2222/tcp                   ALLOW       Anywhere     # SSH administration
80/tcp                     ALLOW       Anywhere     # HTTP - WordPress
21/tcp                     ALLOW       Anywhere     # FTP - canal de controle
40000:40100/tcp            ALLOW       Anywhere     # FTP - mode passif
```

Depuis l'extérieur de la VM :

```console
outsideTheVM:~$ ssh clecart@192.168.56.10 -p 2222
clecart@192.168.56.10's password:
Welcome to Ubuntu ...
clecart@clecart-host:~$ hostname
clecart-host
```

> ⚠️ L'auditeur se connecte en `clecart` **par mot de passe**. C'est pourquoi
> `PasswordAuthentication` reste à `yes` globalement, la restriction par clé
> n'étant appliquée qu'à `luffy` via le bloc `Match User luffy`.

### Utilisateurs

```console
luffy@clecart-host:~$ groups luffy
luffy : luffy sudo
luffy@clecart-host:~$ echo ~ ; echo $HOME
/home/luffy
/home/luffy

zoro@clecart-host:~$ sudo cat /etc/shadow
zoro is not in the sudoers file.  This incident will be reported.
zoro@clecart-host:~$ groups zoro
zoro : zoro
zoro@clecart-host:~$ echo $HOME
/home/zoro
```

Puis l'épreuve `kratos` : voir
[§14](#14-mémo-audit--créer-lutilisateur-kratos-en-moins-de-10-minutes).

### Services

L'auditeur dépose un fichier témoin, puis le récupère en FTP :

```console
$ sudo touch /backup/audit-check

$ ftp 192.168.56.10
Name: nami
230 Login successful.
ftp> ls
-rw-r--r--    1 0   0    0 Aug 25 15:00 audit-check
ftp> get audit-check
226 Transfer complete.
```

> Vérifié : `sudo touch` crée le fichier en `644 root:root`. `/backup` étant en
> `750 root:nami`, `nami` peut **traverser et lister** le dossier (bit `x` du
> groupe) puis **lire** le fichier (bit `r` des autres). Le téléchargement
> fonctionne, et l'écriture reste impossible.

```console
$ ftp 192.168.56.10
Name: anonymous
331 Please specify the password.
Password:                     # vide
530 Login incorrect.
ftp: Login failed
```

### WordPress

```console
$ curl -I http://192.168.56.10/
HTTP/1.1 200 OK

$ curl -I http://192.168.56.10/wp-config.php
HTTP/1.1 403 Forbidden
```

### Sauvegarde

```console
$ sudo crontab -l
0 0 * * * /usr/local/bin/backup-wordpress.sh
```

Test accéléré mené par l'auditeur — il **vide `/backup` et supprime le journal**,
passe la planification à `* * * * *`, attend une minute :

```console
$ ftp 192.168.56.10            # user nami
ftp> ls
-rw-r----- 1 0 1004  412K Aug 25 15:01 wordpress-2026-08-25_15-01-01.tar.gz
ftp> get wordpress-2026-08-25_15-01-01.tar.gz
226 Transfer complete.

$ cat /var/log/backup.log
2026-08-25 15:01:03 - SUCCESS - wordpress backup created!, date: 2026-08-25 15:01:03, file: wordpress-2026-08-25_15-01-01.tar.gz, size: 412K, duration: 2s
```

**Ne pas oublier de remettre `0 0 * * *`** à la fin du test.

---

## 18. Glossaire des commandes

### Système et services

| Commande | Rôle |
|---|---|
| `sudo` | Exécute une commande avec des privilèges élevés, de façon ciblée et tracée (`/var/log/auth.log`) |
| `systemctl start\|stop\|restart\|reload` | Pilote un service. `reload` relit la configuration sans couper les connexions en cours ; `restart` coupe puis relance |
| `systemctl enable\|disable` | Active ou désactive le démarrage automatique au boot |
| `systemctl status` | État d'un service, PID, et dernières lignes de journal |
| `journalctl -u service -f` | Journal d'un service, `-f` pour suivre en direct |
| `hostnamectl` | Lit et définit le nom de la machine |
| `timedatectl` | Fuseau horaire et synchronisation NTP |

### Fichiers et permissions

| Commande | Rôle |
|---|---|
| `chmod` | Modifie les droits. Octal : `4`=lire, `2`=écrire, `1`=exécuter, dans l'ordre propriétaire/groupe/autres |
| `chown user:group` | Change le propriétaire et le groupe. `-R` = récursif |
| `ls -l` / `ls -ld` | Détail d'un fichier / du **dossier lui-même** plutôt que de son contenu |
| `find chemin -type f -exec cmd {} \;` | Parcourt une arborescence et exécute une commande sur chaque résultat |
| `tar -czf` / `-xzf` / `-tzf` | Crée / extrait / liste une archive compressée |
| `tee` | Écrit sur la sortie standard **et** dans un fichier ; `-a` pour ajouter. Permet d'écrire dans un fichier root derrière un `sudo` |

### Utilisateurs

| Commande | Rôle |
|---|---|
| `adduser` | Création complète d'un compte (Debian/Ubuntu) |
| `usermod -aG groupe user` | Ajoute à un groupe secondaire sans effacer les autres |
| `passwd user` | Change un mot de passe |
| `id` / `groups` | Affiche UID, GID et appartenances |
| `getent passwd\|group` | Interroge la base des comptes ou des groupes |
| `sudo -l -U user` | Liste ce qu'un utilisateur a le droit d'exécuter via sudo |

### Réseau

| Commande | Rôle |
|---|---|
| `ip -br a` / `ip route` | Adresses IP / table de routage |
| `netplan generate\|try\|apply` | Valide / applique avec retour arrière / applique la configuration réseau |
| `ss -tulnp` | Sockets en écoute avec les processus propriétaires |
| `ping` | Teste la connectivité IP (protocole ICMP) |
| `ufw allow\|deny\|status` | Gestion du pare-feu |
| `curl -I url` | Récupère uniquement les en-têtes HTTP d'une réponse |

### Bases de données

| Commande | Rôle |
|---|---|
| `mysql` | Client SQL interactif |
| `mysqldump` | Export logique d'une base vers un fichier SQL |
| `SHOW GRANTS FOR 'u'@'h'` | Affiche les privilèges réels d'un compte |

### Diagnostic

| Commande | Rôle |
|---|---|
| `sshd -t` / `apache2ctl configtest` / `netplan generate` | Valident une configuration **avant** de l'appliquer |
| `sshd -T` | Affiche la configuration SSH **effective**, inclusions résolues |
| `grep -ri motif dossier` | Recherche récursive, insensible à la casse |
| `df -h` / `du -h` | Espace libre par système de fichiers / taille d'un fichier ou dossier |
| `lsblk -f` | Arborescence des disques et partitions |

---

## Sources

- [Ubuntu Server Documentation](https://documentation.ubuntu.com/server/)
- [Netplan Reference](https://netplan.readthedocs.io/)
- [OpenSSH — sshd_config(5)](https://man.openbsd.org/sshd_config)
- [vsftpd.conf(5)](https://security.appspot.com/vsftpd/vsftpd_conf.html)
- [MySQL 8.0 Reference Manual — Securing the Initial Account](https://dev.mysql.com/doc/refman/8.0/en/)
- [WordPress — Hardening WordPress](https://developer.wordpress.org/advanced-administration/security/hardening/)
- [UFW — Ubuntu Community Help](https://help.ubuntu.com/community/UFW)
