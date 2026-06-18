# Proxmox - Post install

## Remove popup and subscription
Visit here and follow
[Community PVE scripts](https://community-scripts.org/scripts/post-pve-install?id=post-pve-install)

## Network

- Create a separate network bridge for VM

On the GUI

`Node > Network > Create > Linux Bridge > bridge port > enXXXX`

- Disable Wake up on Lan

Edit `/etc/network/interfaces`

```
auto lo
iface lo inet loopback

# Disable Wake Up On Lan
up ethtool -s nic0_lm  wol d
up ethtool -s nic1_10g wol d
up ethtool -s nic2_10g wol d

iface nic0_lm inet manual

iface nic1_10g inet manual

iface nic2_10g inet manual

auto vmbr0
iface vmbr0 inet static
        address 192.168.40.10/24
        gateway 192.168.40.1
        bridge-ports nic0_lm
        bridge-stp off
        bridge-fd 0
#Bridge for admin

auto vmbr1
iface vmbr1 inet manual
        bridge-ports nic1_10g
        bridge-stp off
        bridge-fd 0
#Bridge for VM

source /etc/network/interfaces.d/*
```

to reload type `ifreload -av`

## key for ssh & disable password for ssh
Create a key pair and put it on `~/.ssh/` in the remote machine
On proxmox, append (or replace, more secure) the content of the file `~/.ssh/authorized_keys` (symlink) with the public key

Then to cancel the password login, create a new file in `/etc/ssh/sshd_config.d/disable_password_auth.conf` and place:

```
PermitRootLogin prohibit-password
```

## Reduce GRUB time

Edit the file `/etc/default/grub`

```
GRUB_DEFAULT=0
GRUB_TIMEOUT=1
GRUB_DISTRIBUTOR=`( . /etc/os-release && echo ${NAME} )`
GRUB_CMDLINE_LINUX_DEFAULT="quiet"
GRUB_CMDLINE_LINUX=""
```
then run `update-grub`

## Install NUT Client
- Use Shell on Proxmox Node to install NUT Client

```
apt-get install nut-client
```

- Edit `/etc/nut/nut.conf` 

Modify the last line

```
MODE=netclient
```

- Edit `/etc/nut/upsmon.conf`

```
MONITOR eaton5p1550i@192.168.40.15:3493 1 ups_user p@ssw0rd secondary

NOTIFYCMD /usr/sbin/upssched
SHUTDOWNCMD "/sbin/shutdown -h now"

NOTIFYFLAG ONBATT SYSLOG+EXEC
NOTIFYFLAG LOWBATT SYSLOG+EXEC
NOTIFYFLAG ONLINE SYSLOG+EXEC
NOTIFYFLAG COMMBAD SYSLOG+EXEC
NOTIFYFLAG COMMOK SYSLOG+EXEC
NOTIFYFLAG REPLBATT SYSLOG+EXEC
NOTIFYFLAG NOCOMM SYSLOG+EXEC
NOTIFYFLAG FSD SYSLOG+EXEC
NOTIFYFLAG SHUTDOWN SYSLOG+EXEC
```

- Edit `/etc/nut/upssched.conf`

```
CMDSCRIPT   /etc/nut/upssched-cmd
PIPEFN      /var/run/nut/upssched.pipe
LOCKFN      /var/run/nut/upssched.lock

AT NOCOMM   * EXECUTE NOTIFY-NOCOMM
AT COMMBAD  * START-TIMER NOTIFY-COMMBAD 10
AT COMMOK   * CANCEL-TIMER NOTIFY-COMMBAD COMMOK
AT FSD      * EXECUTE NOTIFY-FSD
AT LOWBATT  * EXECUTE NOTIFY-LOWBATT
AT ONBATT   * EXECUTE NOTIFY-ONBATT
AT ONLINE   * EXECUTE NOTIFY-ONLINE
AT REPLBATT * EXECUTE NOTIFY-REPLBATT
AT SHUTDOWN * EXECUTE powerdown
AT ONBATT   * START-TIMER powerdown 60
AT ONLINE   * CANCEL-TIMER powerdown
```


- Create and Edit `/etc/nut/upssched-cmd`

```
#! /bin/sh
#

case $1 in
    powerdown)
	logger -t upssched-cmd "Power down by UPS"
	logger -t upssched-cmd "Shutting down PVE"

	/lib/nut/upsmon -c fsd
	;;
    *)
	logger -t upssched-cmd "UPS event $1"
	;;
esac
```

make it executable `chmod +x ./upssched-cmd`

- Enable monitoring services to start up with the machine:
```
sudo systemctl enable nut-monitor
```

- Start monitoring:
```
sudo systemctl start nut-monitor
```

- Verify monitoring:
```
systemctl status nut-client
systemctl status nut-monitor
```

- To check UPS status from your Proxmox Server
`upsc <name>@<address>` for the remote UPS
```
upsc eaton5p1550i@192.168.40.15:3493
```
This will return a bunch of your UPS info if everything is working correctly

# Set color for prompt and ls in ssh

Edit `~/.bashrc` and uncomment
```
# colorized prompt
PS1='${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$'

# You may uncomment the following lines if you want `ls' to be colorized:
export LS_OPTIONS='--color=auto'
eval "$(dircolors)"
alias ls='ls $LS_OPTIONS'
alias ll='ls $LS_OPTIONS -l'
alias l='ls $LS_OPTIONS -lA'
```
To run the `.bashrc` script, just type `. ~/.bashrc`


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

# Truenas SMB share for backup

## create storage on proxmox

On the GUI, go to
`Datacenter > Storage > Add > SMB/CIFS`

```
ID: TrueNAS_SMB
Server : 192.168.XX.XX
Username: proxmox
Password: @El..
Share: backup_proxmox
Enable: checked
Content: only backup (remove Disk Image)
Subdirectory: /Proxmox

Backup retention: Keep all backup
```
## create backup

On the GUI, go to
`Datacenter > Backup > Add `

***

# Move VM from one Node to another

- Power off VM

- Make pre-migration backup using GUI `Node > VM > Backup` and select NAS storage

- Copy to backup 3 files on a shared folder of backup for Node B.

- Find VM config file folder on both nodes
`/etc/pve/nodes/[node name]/qemu-server/110.conf`

- Create a directory to store the config files on a shared mount point, accessible by Node B (useful as I was migrating several VMs at once)
`mkdir /mnt/pve/[shared]/pve-conf-files`

 - Copy the config file from Node A to the config folder on the shared drive
`cp 110.conf.bak /mnt/pve/[shared]/pve-conf-files`

- Copy back this config file on Node B
`cp /mnt/pve/[shared]/pve-conf-files/.110.conf.bak .`

- On the GUI, navigate to te VM and go to `Node > VM > Backup`, select the backup and click `Restore`.

- If required copy the ISO `cp /var/lib/vz/template/iso/filename.iso localfilename.iso`

- Test boot on new node.

- Make post-migration backup.

***

## Check size on the disk of files

```
apt-get install ncdu
ncdu -x /
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

# Install Home Assistant VM

## Obtain the VM image

- Navigate to the installation page on the HA website: [Home Assistant Install Alternative](https://www.home-assistant.io/installation/alternative) 

- Simply right-click the KVM/Proxmox link and copy the address

- In your Proxmox console, navigate to `/var/lib/vz/template/iso` use `wget` to download the file

```bash
wget https://github.com/home-assistant/operating-system/releases/download/17.3/haos_ova-17.3.qcow2.xz
```

- Expand the compressed image (letting a copy of the compressed file)

```bash
xz -d -k haos_ova-17.3.qcow2.xz 
```

## Create the VM

General:
- Select your VM name and ID
- Select 'start at boot'

OS:
- Select 'Do not use any media'

System:
- Change 'machine' to 'q35'
- Change BIOS to OVMF (UEFI)
- Select the EFI storage (typically local-lvm)
- Uncheck 'Pre-Enroll keys'
- Check 'Qemu Agent'

Disks:
- Delete the SCSI drive and any other disks

CPU:
- Set minimum 2 cores

Memory:
- Set minimum 4096 MB

Network:
- Leave default unless you have special requirements (static, VLAN, etc)

Confirm and finish. Do not start the VM yet.

## Add the image to the VM

- In your node's console, use the following command to import the image from the host to the VM

```bash
qm importdisk <VM ID> </path/to/file.qcow2> <EFI location>
```
For example,

```bash
qm importdisk 205 /home/user/haos_ova-12.0.qcow2 local-lvm
```

- Close the node's console and select your HA VM

- Go to the 'Hardware' tab

- Select the 'Unused Disk' and click the 'Edit' button

- Check the 'Discard' box if you're using an SSD then click 'Add'

- Select the 'Options' tab

- Select 'Boot Order' and hit 'Edit'

- Check the newly created drive (likely scsi0) and uncheck everything else

## Finishing

- Start the VM

- Check the shell of the VM. If it booted up correctly, you should be greeted with the link to access the Web UI.

- Navigate to `<VM IP>:8123`

Done. Everything should be up and running now.

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
