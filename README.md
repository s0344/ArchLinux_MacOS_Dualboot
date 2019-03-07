# Dual Boot: Arch Linux and MacOS

This is a guide for Dual Booting Arch Linux on MacOS. All the information are gathered through other guides and digging in google.

All the source will be listed at the end of the guide.

---
## Steps overview
  1. [ Create bootable USB ](#1)
  2. [ Disk partition on Mac ](#2)
  3. [ Partition for Arch linux ](#3)
  4. [ Format Partition ](#4)
  5. [ Mount partitions and create swap ](#5)
  6. [ Installation ](#6)
  7. [ Optimize fstab for SSD, add swap ](#7)
  8. [ Configure system ](#8)
  9. [ Grant sudo right ](#9)
  10. [ Set up locale ](#10)
  11. [ Set up mkinitcpio hooks and run ](#11)
  12. [ Set up GRUB/EFI ](#12)
  13. [ Create boot.efi ](#13)
  14. [ Back to Mac - Configuration ](#14)
  15. [ Install rEFInd ](#15)
  16. [ Back to Arch - Post installation ](#16)
  16. [ Wireless setup ](#17)
---  
<a name="1"></a>
## 1. Create [bootable USB](https://wiki.archlinux.org/index.php/USB_flash_installation_media) with Arch ISO
  - head to link for details
---
<a name="2"></a>
## 2. Disk partition on Mac
  - Open `Disk Utility` on Mac
  - Select the drive to be partitioned in the left-hand column (not the partitions!). Click on the Partition button
  - Add a new partition by pressing the + button and choose how much space you want to leave for OS X, and how much for the new partition. Keep in mind the new partition will be formatted in Arch Linux, so you can choose any partition type you want
  - Boot the Arch installation media by holding down the Alt during boot
---  
<a name="3"></a>
## 3. Partition for Arch linux
  - Use `lsblk` to check the name of your drive
  - Run `cgdisk /dev/sda` (if going to install somewhere else, it may not be /sda)
  - Delete the partition that is created for Arch from last step
  - Create new partition for Arch
  > add 128MB between last apple partition and first Arch partition
  
  > To do it, while creating first partition at the `first sector` part, calculate: 128*2048+(the value on the left inside the parenthesis)
  
  - List of partition that is needed:
  
  Dir | Size | Partition Type (Hex code) | Partition Name
  ----|------|---------------------------|---------------
  /dev/sda3 | 128MB | Apple HFS/HFS+ (af00) | Boot Loader
  /dev/sda4 | 256MB | Linux filesystem  (8300) | Boot
  /dev/sda5 | Twice of RAM | Linux Swap  (8200) | Swap
  /dev/sda6 | Rest of space | Linux filesystem  (8300) | Root
---  
<a name="4"></a>
## 4. Format Partition
  - Format the Boot and Root partition with ext4
  > From now on, in this guide /sda4 and /sda6 refers to Boot and Root. Don't follow the same if you have different number for them.
  ```bash
  mkfs.ext4 /dev/sda4
  mkfs.ext4 /dev/sda6
  ```
---
<a name="5"></a>
## 5. Mount partitions and create swap
  - Create swap(/dev/sda5)
  ```bash
  mkswap /dev/sda5
  swapon /dev/sda5
  ```
  - Mount Root(/dev/sda6)
  ```bash
  mount /dev/sda6 /mnt 
  ```
  - Create directory and mount Boot(/dev/sda4)
  ```bash
  mkdir /mnt/boot
  mount /dev/sda4 /mnt/boot
  ```
  - Create efi directory and mount EFI partition from apple(/dev/sda1)
  > Go back and run `cgdisk` to check if not sure
  
  > EFI will not be explained here, google it if interested
  ```bash
  mkdir /mnt/boot/efi
  mount /dev/sda1 /mnt/boot/efi
  ```
---  
<a name="6"></a> 
## 6. Installation
  - This part requires internet connection. Options:
    * Tethered phone via USB ([Android](https://wiki.archlinux.org/index.php/Android_tethering), [IOS](https://wiki.archlinux.org/index.php/IPhone_tethering))
    * Wired (ethernet adapter)
  - Check if you get internet
  ```bash
  systemctl start dhcpcd
  ping -c 5 google.com
  ```
  - After you make sure you have network, proceed to the installation (add `vim` behind if you know how to use it):
  ```bash
  pacstrap /mnt base base-devel
  ```
---  
<a name="7"></a>
## 7. Optimize fstab for SSD, add swap
  - Generate fstab file (important not to mess with partition after generating this file, may cause slow bootup, if happened update uuid  which can be check with `blkid`)
  ```bash
  genfstab -U -p /mnt >> /mnt/etc/fstab
  ```
  - Modify fstab
  ```bash
  nano /mnt/etc/fstab
  /dev/sda6  /      ext4  defaults,noatime,discard,data=writeback  0 1
  /dev/sda5  /boot  ext4  defaults,relatime,stripe=4               0 2
  /swap      none   swap  defaults                                 0 0
  ```
---  
<a name="8"></a>  
## 8. Configure system
  - input the name you want when you see `myhostname` and `myusername`
  ```bash
  arch-chroot /mnt /bin/bash
  passwd
  echo myhostname > /etc/hostname
  ```
  - May need to remove localtime with `rm` if it already exit
  > Utilize *Tab* key to find your place
  ```bash
  ln -s /usr/share/zoneinfo/America/New_York /etc/localtime
  hwclock --systohc --utc
  ```
  - Add new user and download sudo
  > If sudo is already downloaded, please ignore
  ```bash
  useradd -m -g users -G wheel -s /bin/bash myusername
  passwd myusername
  pacman -S sudo
  ```
--- 
<a name="9"></a>
## 9. Grant sudo right
  - Open the sudoer file
  ```bash
  nano /etc/sudoers
  ```
  - uncomment this line `"%wheel ALL=(ALL) ALL"`
---  
<a name="10"></a>
## 10. Set up locale
  ```bash
  nano /etc/locale.gen
  ```
  - Uncomment `en_US.UTF-8 UTF-8` and the line below
  - Now generate locale file 
  ```bash
  locale-gen
  echo LANG=en_US.UTF-8 > /etc/locale.conf
  export LANG=en_US.UTF-8
  ```
---  
<a name="11"></a>
## 11. Set up mkinitcpio hooks and run
  - Insert `keyboard` after `autodetect`, but usually it is there already
  ```bash
  nano /etc/mkinitcpio.conf 
  ```
  - Run it
  ```bash
  mkinitcpio -p linux
  ```
---  
<a name="12"></a>
## 12. Set up GRUB/EFI
  - Install GRUB
  ```bash
  pacman -S grub-efi-x86_64
  pacman -S efibootmgr
  grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
  ```
  - Configure and generate `grub.cfg`
  ```bash
  nano /etc/default/grub
  ```
  - Find and modify this line `GRUB_CMDLINE_LINUX_DEFAULT="quiet rootflags=data=writeback libata.force=1:noncq"`
  - Add the following to the end of file
  ```bash
  # fix broken grub.cfg gen
  GRUB_DISABLE_SUBMENU=y
  ```
  - Generate `grub.cfg`
  ```bash
  grub-mkconfig -o boot/grub/grub.cfg
  ```
---  
<a name="13"></a>
## 13. Create boot.efi
  > Warning: This guide will use rEFInd to perform dual boot and use an unofficial way to configure it
  
  > Nonetheless, it works fine. :)
  
  - Generate `boot.efi` to the current directory but we **need** it in `root`
  > remember to `cd` all the way back to `root`, neccessary for rEFInd to work
  ```bash
  grub-mkstandalone -o boot.efi -d usr/lib/grub/x86_64-efi -O x86_64-efi --compress=xz boot/grub/grub.cfg
  ```
  > If you decide use apple boot loader, go to source and head to other guides
---  
<a name="14"></a>
## 14. Back to Mac - Configuration
  - Head back to MacOS
  ```bash
  exit
  reboot
  ```
  - Hold down `Alt` when boot up and choose Mac and log in
  - Launch Disk Utility
  - Format (“Erase”) /dev/sda3 using Mac journaled filesystem (The Boot Loader partition that you created in the beginning)
  - Create boot file structure
  > Same as before, disk0sX should be what number you have in MacOS
  ```bash
  cd /Volumes/disk0s3
  mkdir System mach_kernel
  cd System
  mkdir Library
  cd Library
  mkdir CoreServices
  cd CoreServices
  touch SystemVersion.plist
  ```
  - Open that file
  ```bash
  nano SystemVersion.plist
  ```
  - Copy this configuration
  ```
  <xml version="1.0" encoding="utf-8"?>
  <plist version="1.0">
  <dict>
      <key>ProductBuildVersion</key>
      <string></string>
      <key>ProductName</key>
      <string>Linux</string>
      <key>ProductVersion</key>
      <string>Arch Linux</string>
  </dict>
  </plist>
  ```
---  
<a name="15"></a>
## 15. Install rEFInd
  - Download [rEFInd](www.rodsbooks.com/refind/) on your Mac
  - Go to "Getting rEFInd" ->  "A binary zip file"
  - Open Terminal
  - `cd` into `Downloads`
  - Run `refind-install` and `mountesp`
  ```bash
  sudo ./refind-install
  sudo ./mountesp
  ```
  - After `mountesp`, you will see ESP show up as a drive. Open it, go to /EFI/refind/. Open `refind.conf` with `textedit`
  - Uncomment these lines: `scan_all_linux_kernels` and `also_scan_dirs`
  - Hide unused boot options:
  	- Find `dont_scan_volume` add `"Preboot","Root"`
	- Find `dont_scan_dir` add `EFI/GRUB`
	> Actually, it depends on what you see during reboot and configure what you don't want to see here
	
  *Optional: Change theme:*
   - [Official website](http://www.rodsbooks.com/refind/themes.html) for theming rEFInd
   - This guide will use [Minimal theme](https://github.com/EvanPurkhiser/rEFInd-minimal)
   - Locate your refind EFI directory. (The file that we configure before)
   - Create a folder called themes inside it(inside `refind`), if it doesn't already exist
   - Clone/download the whole repository into the themes directory.
   - Enable the theme add the following at the end of refind.conf
   ```
   include themes/rEFInd-minimal/theme.conf
   ```
   - Also modify the `menuentry "Arch Linux"` command (comment out the old one if you want to keep it)
   > We are going to identify Arch manually, this is the part that isn't official. The official one didn't work on mines.
   
   > Remember don't use /sda6 if you have different number for root
   ```
   menuentry "Arch Linux"{
    icon /EFI/refind/themes/rEFInd-minimal/icons/os_arch.png
	volume   "Root"
	loader   /boot.efi
	options  "root=/dev/sda6 ro"  
   }
   ```
 ---
<a name="16"></a>
## 16. Back to Arch - Post installation
  - Now, you can reboot on Mac and you will see the rEFInd bootloader then choose Arch.
  > If you skip theming and don't see Arch in boot loader (Since we manually detect arch during the theming section). Go back and undo the hide unused boot process and reboot again.
  
  - Log in with the username you created earlier then the password.

<a name="17"></a>
## 17. Wireless setup
  - Make sure you have ethernet access or others
  - get the name of the interface
  ```
  ip link
  ```
  - get internet access (in this case ethernet)
  ```
  sudo systemctl start dhcpcd@enp0s25
  ```
  - install drivers then reboot
  ```bash
  pacman -S linux-headers dkms broadcom-wl-dkms iw dialog wpa_supplicant
  reboot
  ```
  - You can connect wifi everytime wifi the following command
  ```
  sudo wifi-menu
  ```
  ---
<a name="18"></a>
## 18. Keyboard backlight
  - change the value to "40"
  ```bash
  sudo nano /sys/devices/platform/applesmc.768/leds/smc\:\:kbd_backlight/brightness
  ```
  
### To Be continued: audio...
  
