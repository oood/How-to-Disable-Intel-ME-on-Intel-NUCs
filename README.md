# How to Disable Intel IME on Intel NUCs

## What is the Intel ME?

The Intel Management Engine (ME) has proprietary hardware independent of the CPU and runs an independent operating system. It has its own CPU, DRAM, and ROM, and it has access to the main CPU, DRAM, and even the entire network, and it is designed to run many of the CPU's additional functions, such as firmware-based TMP and remote management functions. Since Intel ME runs on Ring-3[[1]](#1), this means that this separate piece of hardware can fully control the entire computer, both physically and software. 

There are very few details on Intel ME, but there are some analysis articles worth reading[[2]](#2)[[3]](#3). We're disabling it because it's a huge attack vector, if you don't need the functionality it provides, please disable it.

## Credit

This guide relies on the [me_cleaner](https://github.com/corna/me_cleaner) project, [@dt-zero](https://github.com/dt-zero)'s [modified version](https://github.com/corna/me_cleaner/pull/282#issuecomment-671092362) of me_cleaner, and [this guide](https://github.com/mostav02/Remove_IntelME_FPT) written by [@mostav02](https://github.com/mostav02). Of course, special thanks to Positive Technologies for finding [a way to disable Intel ME by setting the HAP bit](https://www.ptsecurity.com/ww-en/analytics/disabling-intel-me-11-via-undocumented-mode/).


## WARNING

Since this involves flashing the IME firmware, if you do it wrong, or if you flash a corrupted image, your computer may not boot and will need to be repaired using a flash programmer. I don't take any responsibility for this, so make sure you're going to do this before you start and make sure you haven't done anything wrong, and if you get a brick please don't complain to me because there's nothing I can do about it.

## Preconditions

- 8th Gen Intel NUC NUC8i3BE(H/HS/K/...) / NUC8i5BE(H/HS/K/...) / NUC8i7BE(H/HS/K/...) (Other models may be similar, but we don't guarantee, you have to test it yourself, and let everyone know your successes through [pull requests](https://github.com/oood/How-to-Disable-Intel-ME-on-Intel-NUCs/pulls))

- a Phillips screwdriver

- a USB drive

- Windows operating system ([why?](https://github.com/mostav02/Remove_IntelME_FPT#what-is-needed-and-why-not-flashrom))

- [me_cleaner](https://github.com/oood/How-to-Disable-Intel-ME-on-Intel-NUCs/raw/main/files/me_cleaner/) by dt-zero

- Windows with [Python 3](https://www.python.org/downloads/windows/) already installed

- [Intel ME System Tools v12](https://github.com/oood/How-to-Disable-Intel-ME-on-Intel-NUCs/tree/main/files/Intel_ME_System_Tools)

- [`head` (GNU utility)](https://github.com/oood/How-to-Disable-Intel-ME-on-Intel-NUCs/tree/main/files/GNU_utility)

## 1. Update the UEFI BIOS

After flashing the IME partition, you will need to revert the changes every time you update the BIOS, otherwise you may not be able to update the BIOS, so it is best to keep the BIOS up to date before starting. If you already have the latest BIOS firmware installed, please ignore this step.

1. Find your NUC BIOS from [the Intel Download Center](https://www.intel.com/content/www/us/en/download-center/home.html)

2. Update your NUC BIOS by following [the Intel NUC BIOS update guide](https://www.intel.com/content/www/us/en/support/articles/000005636/intel-nuc.html) (the F7 update is recommended)


## 2. Install the Software

- First, install Python for your Windows, you will need version 3.X, you can download it on [the Python website](https://www.python.org/downloads/windows/)

- Download dt-zero's modified [me_cleaner](https://github.com/oood/How-to-Disable-Intel-ME-on-Intel-NUCs/raw/main/files/me_cleaner/)

- Download the [Intel ME System Tools v12](https://github.com/oood/How-to-Disable-Intel-ME-on-Intel-NUCs/tree/main/files/Intel_ME_System_Tools), which we will use to backup and flash the IME in Windows. Unzip it to the root directory of the C drive. there should be no spaces or special symbols in the path

- Download the [`head.exe`](https://github.com/oood/How-to-Disable-Intel-ME-on-Intel-NUCs/tree/main/files/GNU_utility) program provided by [GNU utilities](http://unxutils.sourceforge.net/), which we will use to split files later

## 3. Make a Backup

You should have unzipped the Intel ME System Tools to the root of C drive, now we open the Command Prompt (not PowerShell) with administrator privileges[[4]](#4) and type the following command:

### Enter the directory where `FPTW64.exe` is located

````
cd "C:\Flash Programming Tool\WIN64"
````

Note: Here we use a 64-bit system, if you use a 32-bit system, modify `WIN64` to `WIN32`.


### Backup the Intel Flash Descriptor

````
fptw64.exe -DESC -D ifd.bin
````

This is the region we will be modifying and flashing, always keep the original Intel Flash Descriptor image, if you are interested in what it does please read the references[[5]](#5). After disabling the IME, please flash back the original IFD before each BIOS upgrade to avoid BIOS update failure. However, after each BIOS update, you must re-backup all images and then recreate the modified IFD image according to this guide. Never flash a different version of the image.


### Backup the Intel Management Engine

````
fptw64.exe -ME -D ime.bin
````

The first 4KB of the IME is the IFD that was just backed up, but since the `me_cleaner` script does not support directly modifying the IFD, we need to modify the IME and extract the first 4KB of the modified IME.

### Backup the BIOS

````
fptw64.exe -BIOS -D bios.bin
````

Although we don't touch the BIOS partition, but it's good to make a backup, just in case you get a brick, you can flash this backup with a flash programmer.


## Backup full image

````
fptw64.exe -D full.bin
````

This one contains the BIOS, IFD and IME, this is a full dump of the flash, we won't be using it, but just in case, I hope you'll never use it.


### Backup Twice

Every image should be backed up again and check that the file hashes are exactly the same, because if you back up a broken image and flash the broken image in, you're bound to get a brick.

````
fptw64.exe -DESC -D ifd2.bin
fptw64.exe -ME -D ime2.bin
fptw64.exe -BIOS -D bios2.bin
fptw64.exe -D full2.bin
````

Use the following command to check whether the SHA 1 of the two backups are exactly the same

````
certutil -hashfile .\ifd.bin SHA1
certutil -hashfile .\ifd2.bin SHA1

certutil -hashfile .\ime.bin SHA1
certutil -hashfile .\ime2.bin SHA1

certutil -hashfile .\bios.bin SHA1
certutil -hashfile .\bios2.bin SHA1

certutil -hashfile .\full.bin SHA1
certutil -hashfile .\full2.bin SHA1
````

**Now copy these files to your USB flash drive, why? Because after a failed refresh, you still have a chance to save your computer with these files. a USB flash drive is a safe place to prevent you from losing access to your computer's drive.**


## 4. Modify Image to Disable IME

Because the Intel NUC has Intel Boot Guard enabled, and cannot be disabled, the IME region cannot be refreshed because this protection exists, but the IFD region allows the refresh, and the `HAP/AltMeDisable` bit we need to modify is in the IFD region.

You may ask what is HAP, it is the High Assurance Platform (HAP) Program of the US government. HAP is designed to provide trusted computers for US government agencies including NSA[[6]](#6). After HAP mode is enabled, most of the functions of the IME will be disabled, but the power management functions that the IME is responsible for will be retained. There is very little disclosure about HAP on the Internet, but it is certain that the US government does not trust the IME on their computers, and Intel has provided disabling measures for this.

On Intel computers, pressing the power button and turning the computer on is the job of the IME, so HAP mode is designed to keep the computer's basic functionality while disabling all other IME functions. So it's not really disabling the IME, just disabled many features. of course, if the IME is really disabled, the computer won't boot[[7]](#7).

We can enable HAP mode by simply adjusting the `HAP/AltMeDisable` bit located in the IFD area. and we use [@dt-zero](https://github.com/dt-zero) modified [me_cleaner](https://github.com/oood/How-to-Disable-Intel-ME-on-Intel-NUCs/raw/main/files/me_cleaner/) to make this change. since me_cleaner is not used to modify the IFD, but the entire IME (IFD is the first 4KB of the IME), we also need to extract the IFD from the modified IME after modification.

### i. Copy `full.bin` to the same directory as `me_cleaner.py`

### ii. Open a command prompt and run the command below to generate the modified IME image[[8]](#8):

````python
python me_cleaner.py --soft-disable-only --output hap-ime.bin ./full.bin
````

Specify your actual path to `me_cleaner.py` and original `full.bin` in the command above, It's the IME (through full.bin) that needs to be modified here, not the IFD!

### iii. Extract the IFD from the modified IME using `head.exe`

Use the following command in the command prompt

````
head.exe -c 4096 hap-ime.bin > hap-ifd.bin
````

Now we have an IFD image that we can use to flash

### iv. Generate twice and check whether the files hash is consistent

````
python me_cleaner.py --soft-disable-only --output hap-ime2.bin ./full.bin
head.exe -c 4096 hap-ime2.bin > hap-ifd2.bin

certutil -hashfile .\hap-ime.bin
certutil -hashfile .\hap-ime2.bin

certutil -hashfile .\hap-ifd.bin
certutil -hashfile .\hap-ifd2.bin
````

If everything is done, now you have the image ready for flashing.

## 5. Suspend BitLocker Protection in Windows

If you have BitLocker Disk Encryption enabled, please suspend the protection before going to the next step. otherwise please ignore this step.

Intel ME includes a feature called Intel Platform Trust Technology (Intel PTT)[[9]](#9), which allows devices without TPM hardware to use IME firmware-based TPM features, allowing the use such as BitLocker disk encryption, but the TPM feature will not work when the IME is disabled, this means that the disk will not be decrypted automatically,  however you can use the USB flash drive as a way to read the key each time you power on, and in [the last step](#8-the-last-step) I will show you how to do it.

### Suspend BitLocker protection [[10]](#10):

1. Open Control Panel.

2. Select System and Security > BitLocker Drive Encryption > Suspend protection.

3. Select Yes.

**To be on the safe side, I also recommend backing up your BitLocker Recovery Keys to USB.**


## 6. Make the Flash Chip Writable

Use shorting two pins on the motherboard's audio chip (HDA) to unlock the flash so that it can be written to[[5]](#5)[[11]](#11). If you enable BitLocker, after each failure to short the pins, BitLocker will be enabled again, you have to suspend BitLocker again, otherwise you will be locked out of the system after a successful short.

### i. Now completely power off the computer (S5 state)

Hold down the `shift` and click `Shutdown` in the start menu, this will shut down the power completely.

### ii. Disassemble the NUC and take out the motherboard

iFixit has [a very detailed guide](https://www.ifixit.com/Guide/Intel+NUC8i7BEH+Disassembly/131331) on how to take apart the 8th Gen NUC.

Next, remove the fan without removing the CPU cooling copper pipe.

### iii. Ready to start the system

Install the computer's drives, DRAM and plug in the power and video-out cables. There will be a solid green light on the motherboard, which is normal. but do not turn on the power now

### iv. Prepare tweezers or wire or similar to short the 1 and 5 pins of the audio chip in the picture
![short_pins](https://github.com/oood/How-to-Disable-Intel-ME-on-Intel-NUCs/raw/main/files/picture/short_pins.jpg)

### v. Now keep the short circuit and press the power button, the power light is on for 3-5 seconds and then disconnect the short circuit

You can try the next step. If the short is unsuccessful, you will not be able to flash in. and you will need to redo the steps of [Suspend BitLocker](#5-suspend-bitlocker-protection-in-windows), [Completely shutdown the computer, short and turning it on](#6-make-the-flash-chip-writable).

Be patient, you may need to try many times until you succeed with the short circuit. A magnifying glass may help.


## 7. Flash the `hap-ifd.bin`

Enter the following command in the command prompt to rewrite the modified IFD

````
fptw64.exe -DESC -F hap-ifd.bin
````

Do not flash the `hap-ime.bin` (IME), as there is Intel Boot Guard protection, we cannot determine whether it is safe to change the IME. Always just flash the 4KB IFD, not the IME.

If the flash is successful, you're almost done! if it fails, repeat the above steps ([5. Suspend BitLocker Protection in Windows](#5-suspend-bitlocker-protection-in-windows), [6. Make the Flash Chip Writable](#6-make-the-flash-chip-writable) and [7. Flash the hap-ifd.bin](#7-flash-the-hap-ifdbin)).



## 8. The Last Step

Shut down the NUC now, Assemble your NUC back and put the motherboard back, Now let's turn on the NUC，the first boot after disabling the IME may be slow, this is normal.

### Check if it is disabled

- You can go into the BIOS to check the IME version, if it shows 0.0.0.0 it means disabled.

- Or use `MEInfo` in command prompt to query

````
MEInfoWin64.exe -verbose
````

If you plan to continue using Windows and BitLocker, follow [this How to Geek guide](https://www.howtogeek.com/howto/6229/how-to-use-bitlocker-on-drives-without-tpm/) to set up BitLocker without a TPM, you will need a USB drive with BitLocker key to be plugged into your computer every time you turn it on.


Leave the computer on for now. see if your NUC can be exempted from the limitation of automatic shutdown after more than 30 minutes after modifying the BIOS/IME. If it is still not automatically shut down after 30 minutes, then everything is normal, congratulations that you have completed all the settings.

Please remember, before updating the BIOS firmware in the future, please flash back the original `ifd.bin` you backed up and then update, yes, every flashing needs to be short and disassembled. After an update you still need to disable the IME just keep following this guide and don't forget to always regenerate all images after an update, flashing an older version of the image will bring a brick. 

If you are successful, please share your experience and process in the [discussion](https://github.com/oood/How-to-Disable-Intel-ME-on-Intel-NUCs/discussions). Help improve this guide with [Issues](https://github.com/oood/How-to-Disable-Intel-ME-on-Intel-NUCs/issues) and [PRs](https://github.com/oood/How-to-Disable-Intel-ME-on-Intel-NUCs/pulls). If you want to translate this guide, just translate it and submit a PR.

**Enjoy your NUC with IME disabled.**


# References

<a id="1">[1]</a>
[Intel Management Engine - Wikipedia](https://en.wikipedia.org/wiki/Intel_Management_Engine)
> Privilege rings for the x86 architecture. The ME is colloquially categorized as ring −3, below System Management Mode (ring −2) and the hypervisor (ring −1), all running at a higher privilege level than the kernel (ring 0)...

<a id="2">[2]</a>
[What is the Intel ME? - Bitkeks Blog](https://bitkeks.eu/blog/2017/12/the-intel-management-engine.html)
> The Intel Management Engine (ME) is a hardware chip embedded on Intel motherboards in addition to the main CPU. The chip is integrated in most modern Intel chipsets. The ME is most commonly mistaken as vPro or AMT, but is in fact the hardware chip for out-of-band (OOB) control which other Intel vPro software products use for their purpose. One of these implementations using the Intel ME is the Advanced Management Technology (AMT). The ME also exists for Intels server boards, named the Server Platform Services (SPS), and for Atom System on a Chip (SoC) chips, called the Security Engine (SEC)...

<a id="3">[3]</a>
[Neutralize ME firmware on SandyBridge and IvyBridge platforms - Hardened GNU/Linux](https://hardenedlinux.github.io/firmware/2016/11/17/neutralize_ME_firmware_on_sandybridge_and_ivybridge.html)
> First introduced in Intel’s 965 Express Chipset Family, the Intel Management Engine (ME) is a separate computing environment physically located in the (G)MCH chip (for Core 2 family CPUs which is separate from the northbridge), or PCH chip replacing ICH(for Core i3/i5/i7 which is integrated with northbridge).
> 
> The ME consists of an individual processor core, code and data caches, a timer, and a secure internal bus to which additional devices are connected, including a cryptography engine, internal ROM and RAM, memory controllers, and a direct memory access (DMA) engine to access the host operating system’s memory as well as to reserve a region of protected external memory to supplement the ME’s limited internal RAM. The ME also has network access with its own MAC address through the Intel Gigabit Ethernet Controller integrated in the southbridge (ICH or PCH)...

<a id="4">[4]</a> 
[Start a Command Prompt as an Administrator - Microsoft Docs](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/jj717276(v=ws.11))
> - Click the **Start** charm.
> - Type `cmd`, right-click the **Command Prompt tile**, and then click **Run as administrator**.

<a id="5">[5]</a>
[Guide-How To: Unlock Intel Flash Descriptor Read/Write Access Permissions for SPI Servicing - Win-Raid Forum](https://www.win-raid.com/t3553f39-Guide-Unlock-Intel-Flash-Descriptor-Read-Write-Access-Permissions-for-SPI-Servicing.html)
> The Intel Flash Descriptor (FD) is a data structure that is programmed on the SPI flash chip on all Intel based platforms. It contains information such as space allocated for each region of the flash image, read-write permissions for each region, reserved space for vendor-specific data, chipset configuration parameters and more. The fixed size of the Flash Descriptor is 4 KB (0x1000) and, depending on platform generation, roughly consists of these sections:
>
> - Header: Consists of a 0x16 sized Reset Vector and a 0x4 sized Signature tag 0x5AA5F00F.
> - Map: Pointers to all the descriptor sections as well as the size of each.
> - Component: Information about the number & density of all components, read, write and erase frequencies as well as invalid instructions.
> - Region: Defines the offsets & sizes of all available regions which are Flash Descriptor (FD), BIOS, Management Engine (Engine), Gigabit Ethernet (GbE), Platform Data (PDR), Device Expansion 1, Secondary BIOS, CPU Microcode, Embedded Controller (EC), Device Expansion 2, Innovation Engine, 10 Gigabit Ethernet 1, 10 Gigabit Ethernet 2, Reserved 1, Reserved 2 and Platform Trust Technology (PTT).
> - Master: Contains the hardware security settings for the flash, granting read/write permissions for each region and identifying each master.
> - Chipset Soft Strap: Contains PCH/SoC configurable parameters.
> - CPU Complex Soft Strap: Contains Processor configurable parameters.
> - ROM-Bypass Size: Stores the Engine firmware regions’ debug partition size.
> - Reserved: For future use or FD revisions.
> - VSCC Table: Holds the JEDEC ID and the Engine VSCC information for all the SPI Flash chip(s) supported by the SPI image.
> - Upper Map: Determines the length and base address of the Engine VSCC Table.
> - OEM Section: Reserved for use by the OEM/ODM and 0x100 in size.
> 
> Older platforms used the (community-named) Flash Descriptor v1 which, among others differences, could support up to 8 SPI regions. More modern platforms (>= 100-series or APL) utilize Flash Descriptor v2-3 which can support up to 16 SPI regions.

<a id="6">[6]</a>
[Public mention of the High Assurance Platform Program by the National Security Agency - NSA](https://www.nsa.gov/resources/everyone/digital-media-center/video-audio/information-assurance/assets/files/orlando-2010-transcript.pdf)

<a id="7">[7]</a>
[Disabling Intel ME 11 via undocumented mode - Positive Technologies](https://www.ptsecurity.com/ww-en/analytics/disabling-intel-me-11-via-undocumented-mode/)
> The first process that the kernel creates is BUP, which runs in its own address space in ring-3. The kernel does not launch any other processes itself; this is done by BUP itself, as well as a separate LOADMGR module, which we will discuss later. The purpose of BUP (BringUP platform) is to initialize the entire hardware environment of the platform (including the processor), perform primary power management functions (for example, starting the platform when the power button is pressed), and start all other ME processes. Therefore, it is certain that the PCH 100 Series or later is physically unable to start without valid ME firmware...

<a id="8">[8]</a>
[Disabling IME command on 8th generation NUC using me_cleaner by @Yannik - GitHub](https://github.com/corna/me_cleaner/issues/3#issuecomment-751625453)
> Needs to use #282 with LP-Variant (LP = low power = U suffix).
> 
> `python me_cleaner.py --soft-disable-only --output nuc8-soft-disable-only.rom ../me_cleaner/nuc8.rom`
> 
> successfully removes


<a id="9">[9]</a>
[Trusted Platform Module (TPM) Information for Intel NUC - Intel Support](https://www.intel.com/content/www/us/en/support/articles/000007452/intel-nuc.html)
> Intel® Platform Trust Technology (Intel® PTT) - Intel® Platform Trust Technology (Intel® PTT) offers the capabilities of discrete TPM 2.0. Intel PTT is a platform functionality for credential storage and key management used by Windows 8 , Windows® 10 and Windows* 11. Intel PTT supports BitLocker* for hard drive encryption and supports all Microsoft requirements for firmware Trusted Platform Module (fTPM) 2.0.

<a id="10">[10]</a>
[How to Suspend BitLocker Protection - Microsoft Docs](https://docs.microsoft.com/en-us/troubleshoot/windows-client/windows-security/suspend-bitlocker-protection-non-microsoft-updates)


<a id="11">[11]</a>
[Setting HDA_SDO pin HIGH on Intel High Definition Audio chips - GitHub](https://github.com/mostav02/Remove_IntelME_FPT#server--desktop--laptop)


# License


Originally written by [@oood](https://github.com/oood) in 2022, this guide is licensed under [the CC0 license](https://creativecommons.org/publicdomain/zero/1.0/).

[![CC0](https://licensebuttons.net/p/zero/1.0/88x31.png)](https://creativecommons.org/publicdomain/zero/1.0/)
