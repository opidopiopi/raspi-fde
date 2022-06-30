# raspi-fde
## raspberry pi with full disk encryption and remote unlock

I wanted to have a raspberry pi running raspbian on an encrypted filesystem (everything except /boot) and I wanted to be able to unlock the encryption via ssh.

Ressources I used:  
http://paxswill.com/blog/2013/11/04/encrypted-raspberry-pi/  
https://unix.stackexchange.com/questions/5017/ssh-to-decrypt-encrypted-lvm-during-headless-server-boot/79203#79203

```bash
# raspbian default install
dd if=source.img | pv | sudo dd of=/dev/mmcblk1 bs=4M

# put sd into raspi, start raspi
ssh pi@raspi-ip # pw: raspberry
passwd
sudo su; passwd; exit

sudo apt-get install busybox cryptsetup dropbear
sudo mkinitramfs -v -o /boot/initramfs.gz # creates keys and directories for dropbear
lsinitramfs /boot/initramfs.gz |grep -P "sbin/(cryptsetup|dropbear)"

sudo vim /etc/initramfs-tools/initramfs.conf # add DROPBEAR=y and CRYPTSETUP=y, does not enforce including cryptsetup!?
sudo vim /usr/share/initramfs-tools/hooks/cryptroot # enforce setup="yes" so that cryptsetup is included in initramfs
sudo vim /etc/dropbear/authorized_keys # add your ssh-pubkeys

sudo -E CRYPTSETUP=y mkinitramfs -v -o /boot/initramfs.gz # again so that the settings are applied, check output for dropbear and cryptsetup
sudo vi /boot/config.txt # add: initramfs initramfs.gz followkernel
sudo reboot # test reboot
sudo shutdown -h now

# insert sd into another computer
/sbin/e2fsck -f /dev/mmcblk1p2
/sbin/resize2fs -M /dev/mmcblk1p2
# open gparted and make mmcblk1p2 smaller

sudo dd if=/dev/mmcblk1p2 bs=4M | pv | dd of=raspi-plain-root.img bs=4M
# this should have no effect as it should be as small as possible already, but hey why not?
/sbin/e2fsck -f raspi-plain-root.img
/sbin/resize2fs -M raspi-plain-root.img

# resize second partition to its maximum (gksudo gparted)

sudo cryptsetup luksFormat -c aes-xts-plain64 -s 256 -h sha256 --pbkdf-memory 512000 -y /dev/mmcblk1p2
sudo cryptsetup -v luksOpen /dev/mmcblk1p2 raspi_crypt
dd if=raspi-plain-root.img bs=4M | pv | sudo dd of=/dev/mapper/raspi_crypt bs=4M
sudo e2fsck /dev/mapper/raspi_crypt
sudo resize2fs /dev/mapper/raspi_crypt
sudo e2fsck /dev/mapper/raspi_crypt
mkdir /tmp/piroot
sudo mount /dev/mapper/raspi_crypt /tmp/piroot/
sudo mount /dev/mmcblk1p1 /tmp/piroot/boot/
sudo vim /tmp/piroot/boot/cmdline.txt # Change root=/dev/mmcblk0p2 to root=/dev/mapper/raspi_crypt and add cryptdevice=/dev/mmcblk0p2:raspi_crypt and add ip=:::::eth0:dhcp
sudo vim /tmp/piroot/etc/fstab # change /dev/mmcblk0p2 to /dev/mapper/raspi_crypt
sudo vim /tmp/piroot/etc/crypttab # add raspi_crypt /dev/mmcblk0p2 none luks
sudo umount /tmp/piroot/boot/ /tmp/piroot/
sudo cryptsetup luksClose raspi_crypt

# reboot raspi with lan
ssh root@raspi-ip -o "UserKnownHostsFile=~/.ssh/known_hosts-raspi-dropbear"
/sbin/cryptsetup -v luksOpen /dev/mmcblk0p2 raspi_crypt
ps -eo pid,ppid,comm,args # kill sh that was startet by init (1); exit -> pi boots
ssh pi@raspi-ip
sudo mkinitramfs -v -o /boot/initramfs.gz # check for cryptsetup
sudo reboot
ssh root@raspi-ip -o "UserKnownHostsFile=~/.ssh/known_hosts-raspi-dropbear"
/lib/cryptsetup/askpass "enter luks password: " > /lib/cryptsetup/passfifo # exit, wait for the raspi to boot completely
ssh pi@raspi-ip # this is the "normal" openssh-server, we are done :-)
# after every change of the kernel (kernelupdate) the initramfs has to be recreated, in my case I had to specify the new kernel version, it can be found in this directory (e.g. 4.1.19+): /lib/modules
sudo mkinitramfs -v -o /boot/initramfs.gz <kernelversion>
```
