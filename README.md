# Dual Boot: Arch Linux and MacOS

This is a guide for Dual Booting Arch Linux on MacOS. All the information are gathered through other guides and digging in google.

The mac I am using is MacBook Air (early 2015) and should be working on other versions too.

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
  17. [ Wireless setup ](#17)
  18. [ Keyboard backlight ](#18)
  19. [ Battery Performance ](#19)
  20. [ Thermal Management ](#20)
  21. [ Fan Control ](#21)
  22. [ Trackpad ](#22)
  23. [ Function keys ](#23)
  24. [ Other post installation stuff ](#24)
  25. [ Useful applications ](#25)
  26. [ Source ](#26)
---  
<a name="1"></a>
## 1. Create [bootable USB](https://wiki.archlinux.org/index.php/USB_flash_installation_media) with Arch ISO
  - Head to link for details
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
  > Add 128MB between last apple partition and first Arch partition
  
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
  - Input the name you want when you see `myhostname` and `myusername`
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
  - Uncomment this line `"%wheel ALL=(ALL) ALL"`
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
  > Remember to `cd` all the way back to `root`, neccessary for rEFInd to work
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
  - ALL POST INSTALLATION GUIDE CAN BE DONE ANYTIME. YOU CAN INSTALL WINDOW MANAGER FIRST. 
 ---
<a name="17"></a>
## 17. Wireless setup
  - Make sure you have ethernet access or others
  - Get the name of the interface
  ```
  ip link
  ```
  - Get internet access (in this case ethernet)
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
  - Change the value to "40"
  ```bash
  sudo nano /sys/devices/platform/applesmc.768/leds/smc\:\:kbd_backlight/brightness
  ```
 ---
<a name="19"></a>
## 19. Battery Performance
  - PowerTOP: a tool provided by Intel to enable various powersaving modes in userspace, kernel and hardware, available in the official repositories.
  ```bash
  sudo pacman -S powertop
  ```
  - You may want to put your laptop on battery power and calibrate powertop
  ```bash
  powertop --calibrate
  ```
  - You can create a systemd service that will start powertop’s autotune settings on startup.
  - First, go to the `.service` file
  ```bash
   /etc/systemd/system/powertop.service
  ```
  - Apply the following
  ```bash
  [Unit]
  Description=Powertop tunings

  [Service]
  Type=oneshot
  ExecStart=/usr/bin/powertop --auto-tune
  
  [Install]
  WantedBy=multi-user.target
  And enable it to automatically start at boot time:
  ```
  - Enable service on startup
  ```bash
  systemctl enable powertop.service
  ```
 ---
<a name="20"></a>
## 20. Thermal Management
  - You can do much to save power now. THe most important parts are that you install “thermald” and “cpupower”.
  - Thermald: Thermald is a deamon regulating the CPU speed, when your CPU runs too hot.
  ```bash
  sudo pacman -S thermald
  sudo systemctl enable thermald
  sudo systemctl start thermald
  ```
  - CPUPower: CPUPower can set the governor of your CPU clock speed. So it can set your Intel CPU to powersave mode, which saves a lot energy.
  ```bash
  sudo pacman -S cpupower
  sudo systemctl enable cpupower
  sudo systemctl start cpupower
  ```
  - Set governor to powersave mode
  ```bash
  cpupower frequency-set -g powersave
  ```
 ---
<a name="21"></a>
## 21. Fan Control
  - Standard way of doing it
  ```bash
  sudo pacman -S mbpfan-git
  sudo systemctl mbpfan.service
  ```
  - Since this doesn't work on my device, I spent some time an found another way to do it. (less efficient)
  - I found the standard way is an enhanced version of [this](http://allanmcrae.com/2011/08/mbp-fan-daemon-update/). However, I download this and it doesn't work. At the end, I manually put the `.c` file under `/usr/bin/`, run a command to build, and write a service file for it to run on startup.
  - First download that file in link and unzip it
  ```bash
  tar xvf mbpfan-1.1-1.src.tar.gz
  ```
  - Use `cp` command to copy file to this directory `/usr/bin/`
  ```bash
  cp filename directory
  ```
  - Modify the values in the file.
  > Change fan speed: For my device, the lowest and highest is 1200 and 6500. (check sys file, google it for more details)
  > Change temperature thersholds: I set it as 50,60,70 because I like it to be lower
  > Change polling interval: I set it to 1 so it will be more responsive
  > Change `get_temp` function since the directory is wrong (for me)
  ```c
  unsigned short get_temp()
  {
	  FILE *file;
	  unsigned short temp;
	  unsigned int t0;

	  file=fopen("/sys/class/thermal/thermal_zone0/hwmon1/temp1_input","r");
	  fscanf(file, "%d", &t0);
	  fclose(file);

	  temp = (unsigned short)(ceil((float)(t0)/1000.));
	  return temp;
  }
  ```
  - After all the modifications, run this to build
  ```bash
  gcc -o mbpfan -lm -Wall $CFLAGS $LDFLAGS mbpfan.c
  ```
  - You can test it now by running the command `mbpfan` in command line
  > remember to stress the machine to see if the fan kicks in
  
  - Set `.service` file for startup
  - Create a `mbpfan.service` file under `/etc/systemd/system/` then copy this
  > Fun fact: `sys` file is virtual. What I mean is it always make a new one on every startup. And since the driver need to modify fan speed value which is located under `sys`, we set the driver to enable after `sys` is created.
  ```bash
  [Unit]
  Description=Fan Control for Macbook
  After=syslog.target
  After=remote-fs.target
  
  [Service]
  User=root
  Type=forking
  Restart=always
  ExecStart=/usr/bin/mbpfan
  
  [Install]
  WantedBy=multi-user.target
  ```
  - Enable it and restart
  ```bash
  sudo systemctl enable mbpfan.service
  restart
  ```
  - Use `list-unit` and find `mbpfan.service` to check if it is running
  ```bash
  sudo systemctl list-unit
  ```
 ---
<a name="22"></a>
## 22. Trackpad
  - Install xf86-input-synaptics
  ```bash
  sudo pacman -S xf86-input-synaptics
  ```
  - Configure `/etc/X11/xorg.conf.d/50-synaptics.conf`
  ```bash
  Section "InputClass"
    MatchIsTouchpad "on"
    Identifier      "touchpad catchall"
    Driver          "synaptics"
    # 1 = left, 2 = right, 3 = middle
    Option          "TapButton1" "1"  
    Option          "TapButton2" "3"
    Option          "TapButton3" "2"
    # Palm detection
    Option          "PalmDetect" "1"
    # Horizontal scrolling
    Option "HorizTwoFingerScroll" "1"
    # Natural Scrolling (and speed)
    Option "VertScrollDelta" "-100"
    Option "HorizScrollDelta" "-100"
  EndSection
  ```
 ---
<a name="23"></a>
## 23. Function keys
  - If your F<num> keys do not work, this is probably because the kernel driver for the keyboard has defaulted to using the media keys and requiring you to use the Fn key to get to the F<num> keys. To change the behavior temporarily, append 2 to `/sys/module/hid_apple/parameters/fnmode`.
  ```bash
  echo 2 > /sys/module/hid_apple/parameters/fnmode
  ```
  - To make the change permanent, set the hid_apple fnmode option to 2:
  ```bash
  /etc/modprobe.d/hid_apple.conf
  options hid_apple fnmode=2
  ```
  - To apply the change to your initial ramdisk, in your mkinitcpio configuration (usually `/etc/mkinitcpio.conf`), make sure you either have `modconf` included in the `HOOKS` variable or `/etc/modprobe.d/hid_apple.conf` in the FILES variable. You would then need to regenerate the initramfs.
	
 ---
<a name="24"></a>
## 24. Other post installation stuff
  - For more post installation stuff, go to this [link](https://medium.com/@philpl/arch-linux-running-on-my-macbook-2ea525ebefe3) starting from topic `Fixing lid closing to suspend` 
 ---
<a name="25"></a>
## 25. Useful applications
  - This is a list of applications, just install it with `pacman` or `yay`
  - Fonts
  > ttf-inconsolata
  
  > ttf-liberation
  
  > noto-fonts
  - File utilities
  > p7zip
  
  > unrar
  - Helpful stuff
  > multilib-devel
  
  > wget
  
  > git
  
  > ranger
  - Install xorg server for screen
  > xorg-server
  
  > xorg-xinit
  - GPU driver
  > Intel: xf86-video-intel
  
  > Nvidia: xf86-video-nvidia
  - GUI program
  > dolphin(another ranger)
  
  > firefox/chromium
  
  > kate(textediter)
  
  > feh(background)
  - i3
  > i3
  
  > konsole(urxvt-unicode)
  
  > dmenu
  ```bash
  echo exec i3 >> ~/.xinitrc
  ```
  - Sound
  > pulseaudio
  
  > pavucontrol
  
  > pulseaudio-alsa
  
  > alsa-utils
  
  > Unmute master channel
  ```bash
  amixer sset Master unmute
  ```
  - Yay (another pacman)
  > git clone <yay git site>
  ```bash
  makepkg -sri
  ```
	
 ---
<a name="26"></a>
## 26. Source
  - https://wiki.archlinux.org/index.php/Mac#Installation
  - http://panks.me/posts/2013/06/arch-linux-installation-with-os-x-on-macbook-air-dual-boot/
  - https://github.com/pandeiro/arch-on-air
  - https://wiki.archlinux.org/index.php/GRUB#Installation_2  (for GRUB command)
  - https://www.youtube.com/watch?v=OyZVhosCwBM
  - https://bbs.archlinux.org/viewtopic.php?id=225409&p=2
  - https://mchladek.me/post/arch-mbp/
 ---

