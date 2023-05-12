H3C版本升级

此次升级设备为：Release Version:    H3C S6520-48S-EI-2429

将版本由`Release 2429`升级到`Release 2432P05`

导入升级文件到文件flash:/内

源版本信息

```shell
<NJ-6520-2>dis version
H3C Comware Software, Version 7.1.045, Release 2429
Copyright (c) 2004-2016 Hangzhou H3C Tech. Co., Ltd. All rights reserved.
H3C S6520-48S-EI uptime is 0 weeks, 0 days, 0 hours, 15 minutes
Last reboot reason : Cold reboot

Boot image: flash:/s6520ei-cmw710-boot-r2429.bin
Boot image version: 7.1.045, Release 2429
  Compiled Apr 28 2016 16:00:00
System image: flash:/s6520ei-cmw710-system-r2429.bin
System image version: 7.1.045, Release 2429
  Compiled Apr 28 2016 16:00:00


Slot 1:
Uptime is 0 weeks,0 days,0 hours,15 minutes
S6520-48S-EI with 2 Processors
BOARD TYPE:         S6520-48S-EI
DRAM:               2048M bytes
FLASH:              512M bytes
PCB 1 Version:      VER.B
Bootrom Version:    153
CPLD 1 Version:     002
CPLD 2 Version:     002
Release Version:    H3C S6520-48S-EI-2429
Patch Version  :    None
Reboot Cause  :     ColdReboot
[SubSlot 0] 48SFP Plus
```



设定系统版本操作

```shell
<NJ-6520-2>boot-loader file flash:/S6520EI-CMW710-R2432P05.ipe all main
H3C S6520-48S-EI images in IPE:
  s6520ei-cmw710-boot-r2432p05.bin
  s6520ei-cmw710-system-r2432p05.bin
This command will set the main startup software images. Continue? [Y/N]:Y
Add images to slot 1.
Decompressing file s6520ei-cmw710-boot-r2432p05.bin to flash:/s6520ei-cmw710-boot-r2432p05.bin.........................Done.
Decompressing file s6520ei-cmw710-system-r2432p05.bin to flash:/s6520ei-cmw710-system-r2432p05.bin............................Done.
Verifying the file flash:/s6520ei-cmw710-boot-r2432p05.bin on slot 1...Done.
Verifying the file flash:/s6520ei-cmw710-system-r2432p05.bin on slot 1.........Done.
The images that have passed all examinations will be used as the main startup software images at the next reboot on slot 1.
Decompression completed.
Do you want to delete flash:/s6520ei-cmw710-r2432p05.ipe now? [Y/N]:Y
```

待以上操作完成后，需要重启设备才能引用新的版本。

```shell
<NJ-6520-2>reboot
Start to check configuration with next startup configuration file, please wait.........DONE!
This command will reboot the device. Continue? [Y/N]:y
Now rebooting, please wait...
%Jan  1 00:15:35:565 2011 NJ-6520-2 DEV/5/SYSTEM_REBOOT: System is rebooting now.

# 以下为系统自动升级过程
Starting......
Press Ctrl+D to access BASIC BOOT MENU

********************************************************************************
*                                                                              *
*                    H3C S6520-48S-EI BOOTROM, Version 153                     *
*                                                                              *
********************************************************************************
Copyright (c) 2004-2016 Hangzhou H3C Technologies Co., Ltd.

Creation Date   : Apr  5 2016,17:24:04
CPU Clock Speed : 1000MHz
Memory Size     : 2048MB
Flash Size      : 512MB
CPLD Version    : 002/002
PCB Version     : Ver.B
Mac Address     : 1CAB34949955


PEX mode is disabled.
Press Ctrl+B to access EXTENDED BOOT MENU...0
Loading the main image files...
Loading file flash:/s6520ei-cmw710-system-r2432p05.bin..........................
................................................................................
................................................................................
................................................................................
................................................................................
................................................................................
................................................................................
................................................................................
....Done.
Loading file flash:/s6520ei-cmw710-boot-r2432p05.bin............................
..................................................................Done.

Extended BootRom Version is not equal,updating? (Y/N):     # 此处会自动选‘Y’
Updating extended BootRom..................Done.
Basic BootRom Version is not equal,updating? (Y/N):        # 此处会自动选‘Y’
Updating Basic BootRom....................Done.
CPLD Version is not equal, Updating? (Y/N):                # 此处会自动选‘Y’
Updating cpld.............................................................................................................................................................................................................Done.
CPLD updated,System is rebooting now.

# 以下为系统启动过程
Starting......
Press Ctrl+D to access BASIC BOOT MENU

********************************************************************************
*                                                                              *
*                    H3C S6520-48S-EI BOOTROM, Version 157                     *
*                                                                              *
********************************************************************************
Copyright (c) 2004-2017 New H3C Technologies Co., Ltd.

Creation Date   : Apr 10 2017,09:41:12
CPU Clock Speed : 1000MHz
Memory Size     : 2048MB
Flash Size      : 512MB
CPLD Version    : 003/002
PCB Version     : Ver.B
Mac Address     : 1CAB34949955


PEX mode is disabled.
Press Ctrl+B to access EXTENDED BOOT MENU...0
Loading the main image files...
Loading file flash:/s6520ei-cmw710-system-r2432p05.bin..........................
................................................................................
................................................................................
................................................................................
................................................................................
................................................................................
................................................................................
................................................................................
....Done.
Loading file flash:/s6520ei-cmw710-boot-r2432p05.bin............................
..................................................................Done.

Image file flash:/s6520ei-cmw710-boot-r2432p05.bin is self-decompressing........
................................................................................
................................................................................
................................................................................
................................................................................
................................................................................
................................................................................
................................................................................
................................................................................
................................................................................
................................................................................
...................................................Done.
System is starting...
Cryptographic algorithms tests passed.

Line aux0 is available.


Press ENTER to get started.

```

查看升级过后的版本信息

```shell
<NJ-6520-2>dis version
H3C Comware Software, Version 7.1.045, Release 2432P05
Copyright (c) 2004-2017 New H3C Technologies Co., Ltd. All rights reserved.
H3C S6520-48S-EI uptime is 0 weeks, 0 days, 0 hours, 2 minutes
Last reboot reason : Warm reboot

Boot image: flash:/s6520ei-cmw710-boot-r2432p05.bin
Boot image version: 7.1.045, Release 2432P05
  Compiled Aug 31 2017 16:00:00
System image: flash:/s6520ei-cmw710-system-r2432p05.bin
System image version: 7.1.045, Release 2432P05
  Compiled Aug 31 2017 16:00:00


Slot 1:
Uptime is 0 weeks,0 days,0 hours,2 minutes
S6520-48S-EI with 2 Processors
BOARD TYPE:         S6520-48S-EI
DRAM:               2048M bytes
FLASH:              512M bytes
PCB 1 Version:      VER.B
Bootrom Version:    157
CPLD 1 Version:     003
CPLD 2 Version:     002
Release Version:    H3C S6520-48S-EI-2432P05
Patch Version  :    None
Reboot Cause  :     WarmReboot
[SubSlot 0] 48SFP Plus

```



