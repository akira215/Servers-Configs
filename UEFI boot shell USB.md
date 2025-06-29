
# HowTo-create-a-UEFI-Shell-Boot-Device

1. format the _USB Flash device_ with FAT32
~~2. create the directories _\EFI\BOOT_ on that formatted device~~
3. download _shell.efi_ from:<br>https://github.com/tianocore/edk2/blob/edk2-stable201903/ShellBinPkg/UefiShell/X64/Shell.efi<BR>
   **NOTE:** Please also find <img src="https://github.com/KilianKegel/pictures/blob/master/New-icon.png"  width="18" height="18">https://github.com/KilianKegel/Visual-UEFI-SHELL to create an *up-to-date* **UEFI Shell**
4. rename _shell.efi_ to _BOOTX64.efi_
5. copy _BOOTX64.efi_ to the boot device into the ~~folder _\EFI\BOOT_~~ root of the USB drive - that's it
6. Restart the PC and enter the BIOS. If you use ASUS UEFI BIOS Utility in advanced mode, mouse click on the Exit (not by using keyboard “Esc”), in the next dialog select “Launch EFI Shell from filesystem device”. Other BIOS should behave similarly.
7. make sure, that on the platform SECURE BOOT is disabled in BIOS Setup!
8. adjust boot order to start UEFI Shell boot device

Nice to have: <br>**find** and **more** MSDOS clone for UEFI Shell:<br>

8. create the directories _\EFI\TOOLS_ on that formatted device<br>
9. download _find.efi_ and _more.efi_ from https://github.com/KilianKegel/Visual-DOS-Tools-for-UEFI-Shell/tree/master/x64/UEFIx86-64%20(Torito%20C%20Library)<br>
10. copy _find.efi_ and _more.efi_ to the boot device into the folder _\EFI\TOOLS_ - that's it

Alternative solution from [**congatec**](https://www.congatec.com/us/support/application-notes/article/an31-how-to-create-a-bootable-usb-stick-with-a-uefi-shell/).

# Flash LSI card from USB

Step-by-step procedure:
1. Insert the controller card in a PCIe slot. (I’ve used the slot Nr. 3. In case of troubles recognizing the card in your desktop PC try different slots.)
2. Boot the PC and prepare the USB stick:
3. In the USB stick create and format a FAT or FAT32 partition >= 10 MB. (I’ve created 500 MB FAT32 partition. I wouldn't recommend large partitions, who knows if the EFI shell will read every big partition.)
4. Create the sub-folders for EFI boot. In the web there are two different structures: /boot/efi and /efi/boot. For time saving I’ve created both groups, it works.
5. Download from Broadcom following packages: Installer_P20_for_UEFI and 9211-8i_Package_P20_IR_IT_Firmware_BIOS_for_MSDOS_Windows and extract them on your PC’s HDD.
6. Copy from the downloaded packages three files to the USB stick root folder:
7. from the first package the file sas2flash.efi (it is in sub-folder /sas2flash_efi_ebc_rel/);
8. from the second package: 2118it.bin (it is in sub-folder /Firmware/HBA_9211_8i_IT/) and mptsas2.rom (it is in sub-folder /sasbios_rel/).
9. Download from Github Tianocore the precompiled UEFI version 1 Shell: Shell_Full.efi. (Only v1 is applicable, later versions are not compatible with the flash tool and end up with the message: “InitShellApp: Application not started from Shell”.)
10. Rename the Shell_Full.efi in ShellX64.efi and copy this file to following three USB stick destinations: root folder, /boot/efi/, /efi/boot/. (Again, there are different advices, for time saving it easier to use all three choices.)
11. The creative part is completed, it's time for action. Restart the PC and enter the BIOS. If you use ASUS UEFI BIOS Utility in advanced mode, mouse click on the Exit (not by using keyboard “Esc”), in the next dialog select “Launch EFI Shell from filesystem device”. Other BIOS should behave similarly.
12. Next you should see starting shell execution, ending with a prompt: "Shell>" (not the "2.0 Shell>"!).
13. Type the command: map –b (+Enter) for listing of available disks. Locate which one is your USB stick. In my case it is the fs6:
"fs6 :Removable HardDisk - … USB(…)"
14. You can break further execution of the map command by q.
15. Switch to the located USB stick by command fsN: (+Enter) (N=6 – in my example = "fs6:", set N to your USB stick ID).
16. Dir shows the file list:
- 2118IT.BIN
- MPTSAS2.ROM
- sas2flash.efi
- <DIR> BOOT
- <DIR> EFI
- ShellX64.efi
17. The action can start. During it the power shall not be brocken!
18. Erase the controller flash memory: sas2flash.efi -o -e 6.
19. Write the new firmware to the flash: sas2flash.efi -o -f 2118it.bin -b mptsas2.rom.
20. After a while you'll see the success message. You can restart the PC and check if the controller BIOS reports the new “IT”-firmware.
21. The card is ready to use.
