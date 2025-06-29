# Proxmox - Gouverneur de CPU

## Récupérer la valeur actuelle

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```

## Visualiser les valeurs possibles

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors
```

## Spécifier une nouvelle valeur

```bash
echo "schedutil" | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

## Appliquer la nouvelle valeur à chaque redémarrage via Cron

```bash
crontab -e
@reboot echo "schedutil" | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor >/dev/null 2>&1
```

***



# Installation d'OMV5 sur Proxmox


## Téléchargement ISO

Proxmox save iso in /var/lib/vz/template/iso/

````bash
cd /var/lib/vz/template/iso
wget -O openmediavault_5.5.11-amd64.iso https://sourceforge.net/projects/openmediavault/files/5.5.11/openmediavault_5.5.11-amd64.iso/download
````
## Création de la VM

Sur la GUI Proxmox Host > Create VM (top right)

#### General
```bash
ID: 140
Name : OMV
Start at boot : Enable
```

#### OS
```bash
Storage: local
ISO Image : openmediavault_7...
Type : Linux
Version : put the last available
```

#### System
```bash
SCSI Controller: VirtIO SCSI (single or note)
Qemu Agent : enable
BIOS: Default (SeaBIOS)
```

#### Disks
```bash
Storage: local-lvm
Disk size (GiB) : 20
Cache: Write back
Discard: Enable
SSD emulation: Enable
```

#### CPU
```bash
Core: 2
```

#### Memory
```bash
Memory (MiB): 4096
Minimum memory (MiB): 2048
```

#### Network
```bash
Bridge: vmbr0
Model: VirtIO
```

## First steps with OMV

Login  admin:openmediavault

Create an admin: Utilisateurs > Utilisateurs > + ...
Add the new user in the `openmediavault-admin` group

logout and login with new user
remove the admin from `openmediavault-admin` group
```bash
deluser admin openmediavault-admin
```

### MAJ système OMV

```bash
apt update && apt full-upgrade -y
```



### Installation qemu-agent

```bash
apt install -y qemu-guest-agent
reboot
```


### Installation des OMV-extra

```bash
wget -O - https://github.com/OpenMediaVault-Plugin-Developers/packages/raw/master/install | bash
```

# Passthrough partition proxmox

Pour cela, aller dans l'interface de Proxmox, dans l'onglet Disks de l'hote :
Repérer la partition que vous souhaitez monter dans votre VM, dans mon cas, c'est /dev/sda3

En SSH, sur l'hote Proxmox, on utilise cette commande pour connaitre l'id de la partition ( PARTUUID ) :

```bash
lsblk -l -o NAME,PARTUUID /dev/sda3
```

Pour monter la partition dans une VM, toujours en ligne de commande sur l'hote, on utilise la commande :
```bash
qm set <id-VM> -scsi1 /dev/disk/by-partuuid/<PARTUUID>
```

<id-VM> est a remplacer par l'id de la VM a laquelle il faut monter cette partition
<PARTUUID> est a remplacer par le PARTUUID de votre partition que nous avons déterminer juste avant
scsi1 est l'emplacement de montage, dans les vm proxmox scsi0 est le 1er emplacement, puis scsi1 , puis scsi2, ..

# Passthrough disque proxmox
Pour un disque

```bash
lsblk -o +MODEL,SERIAL,WWN
ls -l /dev/disk/by-id/
```

Lister les disques par ID :
```bash
find /dev/disk/by-id/ -type l|xargs -I{} ls -l {}|grep -v -E '[0-9]$' |sort -k11|cut -d' ' -f9,10,11,12
```

Hot-Plug/Add physical device as new virtual SCSI disk

```bash
qm set 140 -scsi1 /dev/disk/by-id/ata-Go-Infinity_221025628994
```
```
->  update VM 140: -scsi1 /dev/disk/by-id/ata-Go-Infinity_221025628994
```

Hot-Unplug/Remove virtual disk
```bash
qm unlink 592 --idlist scsi2
```
```
->  update VM 592: -delete scsi2
```

Stop & Restart VM
