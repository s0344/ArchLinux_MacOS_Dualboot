# Dual Boot: Arch Linux and MacOS

This is a guide for Dual Booting Arch Linux on MacOS. All the information are gathered through other guides and digging in google.
All the source will be listed at the end of the guide.

### 1. Create [bootable USB](https://wiki.archlinux.org/index.php/USB_flash_installation_media) with Arch ISO
### 2. Disk partition on Mac
  - Open `Disk Utility` on Mac
  - Select the drive to be partitioned in the left-hand column (not the partitions!). Click on the Partition button
  - Add a new partition by pressing the + button and choose how much space you want to leave for OS X, and how much for the new partition. Keep in mind the new partition will be formatted in Arch Linux, so you can choose any partition type you want
  - Boot the Arch installation media by holding down the Alt during boot

### 3. Partition for Arch linux
  - Run `cgdisk /dev/sda` (if going to install somewhere else, it may not be /sda)
  - Delete the partition that is created for Arch from last step
  - Create new partition for Arch
  > add 128MB between last apple partition and first Arch partition\n
  > To do it, while creating first partition at the `first sector` part, calculate: 128*2048+(the value on the left inside the parenthesis)
  
  - List of partition that is needed:
  
  Dir | Size | Partition Type (Hex code) | Partition Name
  ----|------|---------------------------|---------------
  /dev/sda3 | 128MB | Apple HFS/HFS+ (af00) | Boot Loader
  /dev/sda4 | 256MB | Linux filesystem  (8300) | Boot
  /dev/sda5 | Twice of RAM | Linux Swap  (8200) | Swap
  /dev/sda6 | Rest of space | Linux filesystem  (8300) | Root
  
### 4. Format Partition
  - Format the Boot and Root partition with ext4
  > From now on, in this guide /sda4 and /sda6 refers to Boot and Root. Don't follow the same if you have different number for them.
  ```
  mkfs.ext4 /dev/sda4
  mkfs.ext4 /dev/sda6
  ```
  
### 5. Mount partitions and create swap
  - Create swap(/dev/sda5)
  ```
  mkswap /dev/sda5
  swapon /dev/sda5
  ```
  - Mount Root(/dev/sda6)
  ```
  mount /dev/sda6 /mnt 
  ```
  - Create directory and mount Boot(/dev/sda4)
  ```
  mkdir /mnt/boot
  mount /dev/sda4 /mnt/boot
  ```
  - Create efi directory and mount EFI partition from apple(/dev/sda1)
  > go back and run `cgdisk` to check if not sure
  > efi will not be explained here, google it if interested
  ```
  mkdir /mnt/boot/efi
  mount /dev/sda1 /mnt/boot/efi
  ```
  
### 6. Installation
  - This part requires internet connection. Options:
    * Tethered phone via USB ([Android](https://wiki.archlinux.org/index.php/Android_tethering), [IOS](https://wiki.archlinux.org/index.php/IPhone_tethering))
    * Wired (ethernet adapter)
  - Check if you get internet
  ```
  systemctl start dhcpcd
  ping -c 5 google.com
  ```
  - After you make sure you have network, procede to the installation (add `vim` behind if you know how to use it):
  ```
  pacstrap /mnt base base-devel
  ```
  
### 7. Optimize fstab for SSD, add swap
  - Generate fstab file (important not to mess with partition after generating this file, may cause slow bootup, if happened update uuid  which can be check with `blkid`)
  ```
  genfstab -U -p /mnt >> /mnt/etc/fstab
  ```
  - Modify fstab
  ```
  nano /mnt/etc/fstab
  /dev/sda6  /      ext4  defaults,noatime,discard,data=writeback  0 1
  /dev/sda5  /boot  ext4  defaults,relatime,stripe=4               0 2
  /swap      none   swap  defaults                                 0 0
  ```
  
### 8. Configure system
  - input the name you want when you see `myhostname` and `myusername`
  ```
  arch-chroot /mnt /bin/bash
  passwd
  echo myhostname > /etc/hostname
  ```
  - May need to remove localtime with `rm` if it already exit
  > Utilize *Tab* function to find your place
  ```
  ln -s /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
  hwclock --systohc --utc
  ```
  - Add new user and download sudo
  > if sudo is already downloaded, please ignore
  ```
  useradd -m -g users -G wheel -s /bin/bash myusername
  passwd myusername
  pacman -S sudo
  ```
