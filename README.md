# #Gentoo Systemd Installation
* A guide on installing [Gentoo](https://www.gentoo.org/)(systemd) with the [Arch Linux installer](https://www.archlinux.org/download/).
* All Credit Goes to [SysEng Quick](https://youtu.be/0DppEL0y4zc)
* Original site : [syseng.technoplaza.net](http://syseng.technoplaza.net/files/gentoo-systemd-command-guide_2.txt)
* [Gentoo Wiki](https://wiki.gentoo.org/wiki/Handbook:AMD64/Full/Installation)

# #Introduction
* Download the Arch linux [iso](https://www.archlinux.org/download/) and burn the iso to your cd/pendrive and create a bootable media.
* Use the bootable media and boot into the Arch linux live environment.
* If you are using UEFI, make sure /sys/firmware/efi/efivars/ directory exists.
* For now, this guide is only for UEFI/GPT installation. If you are installing on BIOS/MBR follow the original site.

# #Partition
### --- UEFI/GPT ---
***Partition&emsp;&emsp;Filesystem&emsp;&emsp;&emsp;Size&emsp;&emsp;&ensp;Description***

/dev/sda1&emsp;&emsp;vfat(ef00)&emsp;&emsp;&emsp;512M&emsp;&emsp; Boot/EFI system partition

/dev/sda2&emsp;&emsp;swap(8200)&emsp;&emsp;&emsp;4G&emsp;&emsp;&ensp;Swap partition

/dev/sda3&emsp;&emsp;ext4(8300)&emsp;&emsp;&emsp;100G&emsp;&emsp;Root partition

/dev/sda4&emsp;&emsp;ext4(8300)&emsp;&emsp;&emsp;&ensp;rest&emsp;&emsp; Home partition

# #Setup File Systems
* Format Boot partition
```
mkfs.fat -F32 /dev/sda1
```
* Format Swap partition
```
mkswap /dev/sda2
```
* Format Root partition
```
mkfs.ext4 /dev/sda3
```
* Format Home partition(Optional: Do if necessary. Make sure you know what you are doing.)
```
mkfs.ext4 /dev/sda4
```

# #Mount Partitions
* Enable swap partition
```
swapon /dev/sda2
```
* Mount root partition at /mnt
```
mount /dev/sda3 /mnt
```
* Create home,boot directories at /mnt
```
mkdir -p /mnt/{home,boot}
```
* Mount boot partition 
```
mount /dev/sda1 /mnt/boot
```
* Mount home partition
```
mount /dev/sda4 /mnt/home
```

# #Install Systemd Stage3
* Change directory to /mnt
``` 
cd /mnt
```
* Download the [Systemd Stage3](https://www.gentoo.org/downloads/) tarball 
```
wget https://bouncer.gentoo.org/fetch/root/all/releases/amd64/autobuilds/20201104T214503Z/stage3-amd64-systemd-20201104T214503Z.tar.xz 
```
  * Make sure to download the latest build.
* Or you can use any commandline/graphical browser to download the tarball.

* Unpack the tarball
```
tar xpf stage3-amd64-systemd-* --xattrs-include='*.*' --numeric-owner 
```
* Once finished, remove the tarball. It won't be necessary anymore.
```
rm stage3-amd64-systemd-*
```

# #Setup Portage Rsync Repo
```
mkdir --parents /mnt/etc/portage/repos.conf
```
```
cp /mnt/usr/share/portage/config/repos.conf /mnt/etc/portage/repos.conf/gentoo.conf
```

# #Enter Systemd Namespace
* Remove the root password before entering the systemd namespace
```
sed -i -e 's/^root:\*/root:/' /mnt/etc/shadow
```
* Enter the systemd namespace using systemd-nspawn
```
systemd-nspawn -bD /mnt
```
  
# #Setup Systemd
***--- Set Locale ---***
```
cat << EOF >> /etc/locale.gen
en_US ISO-8859-1
en_US.UTF-8 UTF-8
EOF
```
```
locale-gen
```
```
localectl set-locale LANG=en_US.UTF-8
```
***--- Set Keymap ---***
```
localectl list-keymaps | grep -i us
```
```
localectl set-keymap us
```
***--- Set Hostname ---***
* Replace 'gentoo-systemd' with the hostname you want.
```
hostnamectl set-hostname gentoo-systemd
```
* Add hostname to /etc/hosts 
```
echo "127.0.1.1 gentoo-systemd.localdomain gentoo-systemd" >> /etc/hosts
```
***--- Set Timezone ---***
```
timedatectl list-timezones | grep -i Kolkata
```
```
timedatectl set-timezone Asia/Kolkata
```
```
timedatectl set-ntp true
```
***--- Leave Systemd Namespace ---***
* Leave the systemd namespace. Don't worry, it doesn't shutdown the system.
```
poweroff
```

# #Generate Fstab
* We will use Arch's genfstab to generate the fstab
```
genfstab -U /mnt >> /mnt/etc/fstab
```

# #Chroot into Gentoo
* Mount the necessary filesystems before chrooting into Gentoo
```
mount -t proc none /mnt/proc
```
```
mount --rbind /dev /mnt/dev
```
```
mount --rbind /sys /mnt/sys
```
* Chroot into the Gentoo environment
```
chroot /mnt /bin/bash
```
```
source /etc/profile
```
```
export PS1="(chroot) ${PS1}"
```

# #Setup Portage
```
emerge-webrsync
```
* Optional: Update package repo to current
```
emerge --sync
```
* Optional: Update portage if it is suggested
```
emerge -1 sys-apps/portage
```

# #Configure Portage
* Add native cflag
```
sed -i -e 's/CFLAGS="/CFLAGS="-march=native /' /etc/portage/make.conf
```
* Specify Makeopts
```
echo 'MAKEOPTS="-j4"' >> /etc/portage/make.conf
```
* Specify Grub platforms
```
echo 'GRUB_PLATFORMS="efi-64"' >> /etc/portage/make.conf
```
* Optional : Add your favourite gentoo [mirrors](https://www.gentoo.org/downloads/mirrors/) for better download speed
```
echo 'GENTOO_MIRRORS="https://mirror.yandex.ru/gentoo-distfiles/ http://mirror.yandex.ru/gentoo-distfiles/"' >> /etc/portage/make.conf
```
***OPTIONAL (RECOMMENDED) : Rebuild packages with new USE flags***
```
emerge -auDN @world
```

# #Install Kernel and System Tools
```
emerge gentoo-sources linux-firmware genkernel grub:2 dosfstools
```

# #Configure Systemd Network
* Change the name(enp*) according to your network device name
```
cat <<EOF > /etc/systemd/network/20-wired.network
[Match]
Name=enp*
 
[Network]
DHCP=yes
EOF
```
```
systemctl enable systemd-networkd.service
```
```
systemctl enable systemd-resolved.service
```

# #Configure and Compile the Kernel
```
genkernel --menuconfig all
```
* Select systemd from gentoo specific init systems

# #Configure and Install [Grub](https://wiki.gentoo.org/wiki/Handbook:AMD64/Installation/Bootloader)
```
echo 'GRUB_CMDLINE_LINUX="init=/usr/lib/systemd/systemd"' >> /etc/default/grub
```
```
grub-install --target=x86_64-efi --efi-directory=/boot
```
```
grub-mkconfig -o /boot/grub/grub.cfg
```

# #Set root password
```
passwd
```

# #Reboot into Gentoo system
```
exit
```
```
reboot
```
