# Arch Linux and BSPWM Installation on X1 Carbon 6th Gen
This "guide" is here to help me install Arch Linux on my ThinkPad X1 Carbon 6th Gen with BSPWM. It also covers wiping your Windows 10 installation that comes with the laptop by default (we clear the SSD's memory cells from within the Arch installation media).

Introduction
---

Make sure to consult the [Arch Wiki](https://wiki.archlinux.org/index.php/Installation_guide) every chance you get. It will explain every question you have (this is mostly a reminder for myself to not ask questions on Reddit when the answer is almost always in plain sight).

Also, in order to reduce the need to look back and forth between this guide and the Arch Wiki, I have included every necessary step to get a working Arch Linux installation going. The difference between my installation on my **X1C6** and your installation will only really vary in the later parts of the installation (setting up drivers and what not).

Also, I am doing a very simple setup with UEFI, systemd-boot, and BSPWM. I will not be using a Desktop Environment because I personaly do not like them. If you want to use LVM or make an encrypted setup, follow [this awesome guide](https://github.com/angristan/arch-linux-install) or [this X1C6 guide](https://github.com/ejmg/an-idiots-guide-to-installing-arch-on-a-lenovo-carbon-x1-gen-6#why-you-should-or-should-not-install-arch).

Bootable USB
---

1. Download the ISO from [here](https://www.archlinux.org/download/) using any mirror that is closest to where you live (or use a torrent from the same page).
2. Create a bootable USB using the terminal (you might have to use sudo)

   **Mac**:
   ``` javascript
   diskutil unmountDisk /dev/diskX
   dd bs=4M if=/path/to/iso of=/dev/diskX conv=sync,noerror
   ```
   
   **Linux**:
   ``` javascript
   umount /dev/sdX
   dd bs=4M if=/path/to/iso of=/dev/sdX status=progress oflag=sync
   ```

BIOS
---

1. Change ```Sleep State``` by going to ```Config > Power > Sleep State [Linux]```
2. Disable ```Secure Boot``` by going to ```Security > Secure Boot > Secure Boot [Disabled]```
3. Enable ```Thunderbolt BIOS Assist Mode``` by going to ```Config > Thunderbolt(TM) 3 > Thunderbolt BIOS Assist Mode [Enabled]```
4. Disable ```I/O Port Access``` items by going to ```Security > I/O Port Access``` and setting any unused I/O Devices to ```Disabled```

Installation
---

* Plug in your Bootable USB and press ```Enter > F12``` and choose your USB from the list. Before continuing, change your terminal font IF you have the WQHD Display (like yours truly):

   ``` javascript
   setfont latarcyrheb-sun32
   ```

1. Your **X1C6** Should be using UEFI, but to double check, type:
   ``` javascript
   ls /sys/firmware/efi/efivars
   ```
2. I purchased the ethernet dongle but if you need WiFi, type:
   ```
   wifi-menu
   ```
   And make sure to check that it worked by typing:
   ```
   ping google.com
   ```
3. Setup your timezone and clock by typing:
   ``` javascript
   timedatectl set-ntp true
   timedatectl set-timezone [Country]/[City]
   timedatectl status
   ```
4. At this point, you should consult the afformentioned guides if you wish to use LVM or encryption. If you just want a basic install like me, consider the following:
   * Type ```lsblk``` or ```fdisk -l``` to see a list of your devices. The **X1C6** should be located under ```/dev/nvme0n1```
      * If you are not using an **X1C6** then your disk might be located under /dev/sdX (X corresponding to a, b, c, etc.). If so, then individual partitions will be marked as ```/dev/sdX1```, ```/dev/sdX2```, and so on. For the **X1C6**, this will be ```/dev/nvme0n1p1```, ```/dev/nvme0n1p2```, and so on.
      
   * I prefer to use ```cfdisk``` because it makes more sense to me. You are free to use ```gdisk``` though!
      ``` javascript
      cfdisk /dev/nvme0n1
      ```
      and select GPT from the menu
   
   * Now we will create our partitioning scheme.
   
      |    Partition   |       Size       |       Type       |
      |:--------------:|:----------------:|:----------------:|
      | /dev/nvme0n1p1 | 512M             | EFI Filesystem   |
      | /dev/nvme0n1p2 | Remaining - Swap | Linux Filesystem |
      | /dev/nvme0n1p3 | 2G               | Swap             |
      
      Use the menu at the bottom of ```cfdisk``` to change the type of each partition, and create them in the order above, making sure to leave room for the swap partition (if you even need it. Personally, I just have it there in case but you can always use a swap file). Then, select ```Write``` at the bottom, write ```Yes```, and quit out of ```cfdisk```.
      
   * Let's create the filesystems for these partitions (**_watch out for using the correct partition, e.i. p1 for EFI, p2 for root, and p3 for swap)_**.
      ``` javascript
      mkfs.ext4 /dev/nvme0n1p2
      mount /dev/nvme0n1p2 /mnt
      ```
      
      ``` javascript
      mkfs.fat -F32 /dev/nvme0n1p1
      mkdir /mnt/boot
      mount /dev/nvme0n1p1 /mnt/boot
      ```
      
      ``` javascript
      mkswap /dev/nvme0n1p3
      swapon /dev/nvme0n1p3
      ```
5. Edit your ```/etc/pacman.d/mirrorlist``` to increase download speed for the next step:
   * Use a tool like ```Reflector``` to make this process faster. I did it all by hand though. Essentially, you just want the mirrors with the best connection relative to you at the top of the file (I put all the servers from the USA at the top).
   
6. Install the base system with ```pacstrap``` and use ```base-devel``` for installing things like ```make``` (useful for installing ```st``` or ```dwm```)
   ``` javascript
   pacstrap /mnt base base-devel
   ```
