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

* Change ```Sleep State``` by going to ```Config > Power > Sleep State [Linux]```
* Disable ```Secure Boot``` by going to ```Security > Secure Boot > Secure Boot [Disabled]```
* Enable ```Thunderbolt BIOS Assist Mode``` by going to ```Config > Thunderbolt(TM) 3 > Thunderbolt BIOS Assist Mode [Enabled]```
* Disable ```I/O Port Access``` items by going to ```Security > I/O Port Access``` and setting any unused I/O Devices to ```Disabled```

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
   
      |    Partition   |             Size             |       Type       |
      |:--------------:|:----------------------------:|:----------------:|
      | /dev/nvme0n1p1 | 256M                         | EFI System       |
      | /dev/nvme0n1p2 | Remaining (Account for Swap) | Linux filesystem |
      | /dev/nvme0n1p3 | 2G                           | Linux swap       |
      
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
   pacstrap /mnt base base-devel neovim wpa_supplicant intel-ucode git
   ```
7. Generate the ```fstab``` with ```-U``` to use ```UUID```
   ``` javascript
   genfstab -U /mnt >> /mnt/etc/fstab
   ```
8. Change Root (chroot) into the installation
   ``` javascript
   arch-chroot /mnt
   ```
9. Open ```/etc/pacman.conf``` and uncomment the following
   ``` javascript
   [multilib]
   Include = /etc/pacman.d/mirrorlist
   ```
   and run ```pacman -Sy```
   
Inside the Installation
---

1. Set the timezone and setup the clock (use your own country and city)
   ``` javascript
   ln -sf /usr/share/zoneinfo/America/Denver /etc/localtime
   hwclock --systohc
   ```
2. Open ```/etc/locale.gen``` and uncomment your needed localization
   ``` javascript
   en_US.UTF-8 UTF-8
   ```
   and run ```locale-gen```
   
3. Create the ```/etc/locale.conf``` file and add your localization
   ``` javascript
   LANG=en_US.UTF-8
   LC_COLLATE=C
   ```
4. Open ```/etc/hostname``` and add your hostname
   ``` javascript
   tredoux
   ```
5. Add this to your ```/etc/hosts```
   ``` javascript
   127.0.0.1    localhost
   ::1          localhost
   127.0.1.1    tredoux.localdomain tredoux

* This is where the Arch Installation Wiki ends. If you followed this guide, you also followed the entire Arch Installation Guide, minus some very useful explanations. However, (and this is a reminder to myself) you should not have had the need to look back and forth between this guide and the Wiki unless you wanted extra explanation (I have a tendency to get anal about my setups and want them to be **_PERFECT_** and waste time looking back and forth between the Wiki and this).

Setting Up ```systemd-boot``` and ```intel-ucode```
---

* Before starting, if you are using LVM or encryption, follow the two guides I posted above and ignore the following

1. Install ```systemd-boot``` in our ```/boot``` directory
   ``` javascript
   bootctl install
   ```
2. Setup the boot files
   * Edit ```/boot/loader/loader.conf``` but set the timeout to something like 3 
   ``` javascript
   default arch
   timeout 0
   editor 0
   ```
   * Edit/create ```/boot/loader/entries/arch.conf``` and add the following
      * Replace ```<UUID>``` with the value of running ```blkid -s UUID -o value /dev/nvme0n1p2``` in your terminal.
   ``` javascript
   title Arch Linux
   linux /vmlinuz-linux
   initrd /intel-ucode.img
   initrd /initramfs-linux.img
   options root=UUID=<UUID> quiet loglevel=3 module_blacklist=btusb,iTCO_vendor_support,iTCO_wdt,uvcvideo nowatchdog rw
   ```

Reboot and Installing Necessary Packages
---

At this point, you should have a very minimal installation with your bootloader setup and all the necessary packages needed to simply run Arch Linux. However, it kind of looks boring, considering it is just a terminal. You also won't be able to have proper power management setup at this point. So, let's move on to the next stage of the installation! (This is where we setup ```X.Org```, ```bspwm```, ```tlp```, etc.)

Set your root password
``` javascript
passwd
```

and exit

``` javascript
exit
umount -R /mnt
shutdown now
```

Unplug your installation media and boot into the new installation.

1. Setup WiFi and use ```ip addr``` to get the name of your WiFi interface
   ``` javascript
   ip addr
   ip link set <interface> down
   cp /etc/netctl/examples/wireless-wpa /etc/netctl/<name>
   nvim /etc/netctl/<name>
   ```
   **i. Inside of ```/etc/netctl/<name>```**
   ``` javascript
   Interface: <interface>
   ESSID: <name of WiFi you are connecting to>
   key: <password of WiFi you are connecting to>
   ```
   **ii. Then**
   ``` javascript
   systemctl enable netctl-auto@wlp2s0.service
   ```
2. Enable ```TRIM``` but make sure your SSD supports it first
   ``` javascript
   systemctl enable fstrim.timer
   ```
3. Add your user, uncomment ```%wheel ALL=(ALL) ALL``` with ```visudo```, change password for your user, and remove login for ```root```
   ``` javascript
   useradd -mG wheel cameron
   visudo
   passwd cameron
   passwd -l root
   ```
4. Alter your ```fstab```
   ``` javascript
   UUID=... / ext4 rw,relatime,data=ordered 0 1
   UUID=... /boot vfat  rw,relatime,fmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,utf8,errors=remount-ro  0 2
   UUID=... none  swap  defaults  0 0
   tempfs /tmp  tempfs  defaults,noatime,mode=1777  0 0
   ```
5. Install ```yay``` package manager
   ``` javascript
   git clone https://aur.archlinux.org/yay.git
   cd yay
   makepkg -si
   yay -S yay
   ```
6. Pacman Hook for clearing cache. Create ```hooks``` folder in ```/etc/pacman.d/``` and create the file ```clean_cache.hook``` in the ```hooks``` folder
   ``` javascript
   [Trigger]
   Operation = Remove
   Operation = Install
   Operation = Upgrade
   Type = Package
   Target = *
   
   [Action]
   Description = Keep currently installed and last cache
   When = PostTransaction
   Exec = /usr/bin/paccache -rvk2
   ```
7. Masking ```lvm2-monitor.service``` helped speed up my boot since I was not using LVM
   ``` javascript
   sudo systemctl mask lvm2-monitor.service
   ```
8. Install ```zsh``` and set it up to your liking
   ``` javascript
   sudo pacman -S zsh
   ```
9. Install ```tlp``` and other power saving tools. I set my charging thresholds to ```45%``` and ```85%```
   ``` javascript
   sudo pacman -S tlp tlp-rdw acpi_call tpacpi-bat
   system_ctl enable tlp.service
   system_ctl enable tlp-sleep.service
   system_ctl mask systemd-rfkill.service
   system_ctl mask systemd-rfkill.socket
   ```
9. Install ```Lenovo Throttling Fix```
   ``` javascript
   yay -S lenovo-throttling-fix-git
   sudo systemctl enable --now lenovo_fix.service
   ```
9. Install ```X.Org``` packages
   ``` javascript
   sudo pacman -S xorg-server xorg-xinit xorg-xbacklight xorg-xrandr xorg-xsetroot xorg-xset fontconfig
   yay -S xclip
   ```
9. Setup our fonts by following the instructions at [this link](https://www.reddit.com/r/archlinux/comments/5r5ep8/make_your_arch_fonts_beautiful_easily/)
   
