# allwinner_a13_debian
Guide how to install debian 12 to a13 tablet

# host setup  
```
mkdir /root/debinst_amd64
debootstrap --arch amd64 bookworm /root/debinst_amd64/ http://ftp.us.debian.org/debian
LANG=C.UTF-8 chroot /root/debinst_amd64/ /bin/bash
apt install wget git build-essential dwarves python3 python3-dev python3-setuptools swig libgnutls28-dev libncurses-dev flex bison libssl-dev bc libelf-dev rsync debhelper crossbuild-essential-armhf
```

# kernel
```
cd /root
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.12.9.tar.xz
tar xf linux-6.12.9.tar.xz
cd linux-6.12.9
wget https://raw.githubusercontent.com/frivermen/allwinner_a13_debian/refs/heads/main/.config
ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make menuconfig
# fix bugs
sed -i 's/--unsigned-changes/-d --unsigned-changes/' scripts/Makefile.package
sed -i 's/CONFIG_LOCALVERSION="-ARCH"/CONFIG_LOCALVERSION="-armhf"/' .config
ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make -j`nproc` bindeb-pkg
```

# uboot
```
cd /root
git clone https://source.denx.de/u-boot/u-boot.git --depth=1
cd u-boot/
ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make q8_a13_tablet_defconfig
ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make -j`nproc`
exit # from chroot debinst_amd64
```

# armhf rootfs
```
mkdir /root/debinst_a13
debootstrap --foreign --arch armhf bookworm /root/debinst_a13 http://ftp.us.debian.org/debian
# linux-image.deb
cp /root/debinst_amd64/root/*.deb /root/debinst_a13/root
cp /root/debinst_amd64/root/linux-6.12.9/arch/arm/boot/dts/allwinner/sun5i-a13-q8-tablet.dtb /root/debinst_a13/boot/a13.dtb
cp /usr/bin/qemu-arm-static /root/debinst_a13/usr/bin
wget https://raw.githubusercontent.com/frivermen/allwinner_a13_debian/refs/heads/main/boot.scr
LANG=C.UTF-8 chroot /root/debinst_a13 qemu-arm-static /bin/bash
# set password 1 for root
echo root:1 | chpasswd
mount -t proc proc /proc
cat <<EOF >> /etc/resolv.conf 
namespace 8.8.8.8
namespace 1.1.1.1
EOF
echo "a13" > /etc/hostname
echo "127.0.0.1     a13" >> /etc/hosts
# setup repos
echo "deb http://ftp.us.debian.org/debian bookworm main contrib non-free non-free-firmware" > /etc/apt/sources.list
apt update
# install selfcompiled kernel
apt install linux-base klibc-utils
dpkg -i /root/linux-image-6.12.9-armhf_6.12.9-5_armhf.deb
dpkg -i /root/linux-libc-dev_6.12.9-5_armhf.deb
dpkg -i /root/linux-headers-6.12.9-armhf_6.12.9-5_armhf.deb
mv /boot/vmlinuz-6.12.9-armhf /boot/zImage
# setup ssh server and root access
apt install openssh-server
sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
# fstab
cat <<EOF >> /etc/fstab
/dev/mmcblk0p2  /       ext4    defaults        0       1 
/dev/mmcblk0p1  /boot   vfat    defaults        0       2 
none            /tmp    tmpfs   defaults,noatime,mode=1777      0      0
EOF
# enable login for uart0 of a13
systemctl enable serial-getty@ttyS0.service
# setup wifi (rt8188cus)
apt install wpasupplicant dhcpd firmware-realtek
# exit from chroot debinst_a13
apt clean
exit
umount /root/debinst_a13/proc
```

# sd card
```
# 100M fat + other_space ext4
fdisk /dev/sdd
o
n
p
1
2048
+100M
n
p
<CR>
<CR>
<CR>
w
mkfs.vfat /dev/sdd1
mkfs.ext4 /dev/sdd2
mount /dev/sdd2 /mnt/
mkdir /mnt/boot
mount /dev/sdd1 /mnt/boot
cp -r /root/debinst_a13/* /mnt/
umount -R /mnt
# uboot flash
dd if=/root/debinst_amd64/root/u-boot/u-boot-sunxi-with-spl.bin of=/dev/sdd bs=1024 seek=8
```

# enable microusb as host
```
cp a13.dtb a13-usb-input.dtb
dtc -I dtb -O dts a13.dtb -o a13.dts
sed -i 's/dr_mode = "otg";/dr_mode = "host";/' a13.dts
dtc -I dts -O dtb a13.dts -o a13.dtb
```

# connect to wifi
```
wpa_supplicant -B -i wlan0 -c<(wpa_passphrase MY_WIFI_NAME 'password')
dhcpclient wlan0
```
