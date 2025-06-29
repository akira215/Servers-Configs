# TrueNas installation

## Get the image

```bash
https://www.truenas.com/truenas-community-edition/
```

Warning this image do not work on ventoy.
-> Flash with balenaEtcher

## Install

Disable secure boot (BIOS > Boot > Secure Boot > OS Type : Other OS
Restart with USB stick in the server

Select 1 - Administrative user (truenas_admin)
Warning ! Password will be QWERTY



# Retrieve config:

System > General Settings > Manage Configuration (top-right) > upload file (on the computer)

Mettre le shell en clavier francais
```bash
setxkbmap fr
```


# Adjustements

## Put System dataset to SSD

Move the system dataset pool to SSD
System > Advanced settings > Storage > System Dataset Pool -> Configure

## Administrator

Add a nex user and put auxiliary group `builtin_administrators`.
Log with this new admin group, select truenas_admin, remove from `builtin_administrators` and click on disable password. Additionaly, remove sudo command.

## Disable root password

Credentials > User > root > edit

- Disable password checked
- SSH password unable unchecked
- Allow all sudo commands
- Upload or copy paste a public SSH key

Generating a SSH key pair could be done in Credentials > Backup credentials > SSH Keypair > Add then generate

System > Services > SSH > Edit uncheck Allow Password Authentication

## Set alerts

Put an email in for the admin user (belonging to builtin-administrator group)

System > General Settings > Email > Settings:
 - GMail OAuth
 - From email (put developer email)
 - From name : put the host name of the server

Then Login To GMail and follow



# Consumption

Check states
```bash
powertop --calibrate
---- REBOOT
powertop --calibrate
powertop --auto-tune
powertop
```
Check tab Idle stats > Pkg (HW) shall be above C3. Check on server, disconnecting the network cable for a while !

Check devices that support ASPM
```bash
sudo lspci -vvv | grep "ASPM .*abled"
sudo lspci -vvv | grep "ASPM"
lspci -vv | awk '/ASPM/{print $0}' RS= | grep --color -P '(^[a-z0-9:.]+|ASPM )'
```

Find a driver corresponding on a device using `/sys`
```bash
$  lspci
...
02:00.0 Ethernet controller: Realtek Semiconductor Co., Ltd. RTL8111/8168B PCI Express Gigabit Ethernet controller (rev 01)
$ find /sys | grep drivers.*02:00
/sys/bus/pci/drivers/r8169/0000:02:00.0
```

Optimize realtek drivers
Refer to script NOT WORKING
find ID of te network adapter running `lspci`
```bash
sudo echo 1 > /sys/bus/pci/devices/0000:01:00.0/link/l1_aspm
```
 NOT WORKING

Check ASPM policy
```
cat /sys/module/pcie_aspm/parameters/policy
```

Set the kernel parameter  (warning!)
```
 midclt call system.advanced.update '{ "kernel_extra_options": "pcie_aspm=force" }'
```

Check available governors
```bash
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

Add a postinit command in System Settings - Advanced - Init/Shutdown Scripts:
```bash
echo "performance" | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```




# Apps
```bash
docker ps
docker exec -it <NAMES> <command>
```



# Rename a ZFS pool

So, firstly, you need to export your pool that you want to rename from the GUI. Note its name.
Before you do that, if your system-dataset is on the pool, its a good idea to move it back to a different pool, or perhaps the boot pool.

System → Advanced Settings → Storage → Configure : System Dataset, then select the pool you want to host it, and save.

Once that’s done,
Storage → Pools, the click the cog to the right of the pool you want to rename and choose “Export/Disconnect”

**DO NOT TICK** “Destroy data on this pool?”

It is up to you if you delete the shares, etc, or have to re-configure them when you re-import the pool.

TICK “Confirm Export/Disconnect” then click EXPORT/DISCONNECT

…

Once that is finished fire up a terminal and import the pool into the CLI with the new name.

```
zpool import original_name new_name
```

(replace original_name and new_name, respectively)

You can check your handy work with Then export the pool again
```
zpool status new_name
```

Then export the pool again
```
zpool export new_name
```

Then import it again in the GUI

Pools → Add, import existing

Job Done.

You may need to reconfigure your shares etc.
