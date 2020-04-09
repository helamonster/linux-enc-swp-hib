# linux-enc-swp-hib
Encrypted Swap Device Setup with Hibernation / Suspend/Resume

The following is a work in progress and is not guaranteed to work properly on your system or at all.
Thanks goes to those sites referenced.

```bash
# Encrypted swap with support for hibernation
# Authentication via password and Yubikey in static mode
# Yubikey: https://www.yubico.com/
# https://help.ubuntu.com/community/EnableHibernateWithEncryptedSwap
# https://www.pbworks.net/ubuntu-guide-dropbear-ssh-server-to-unlock-luks-encrypted-pc/
# https://wiki.archlinux.org/index.php/Dm-crypt/Swap_encryption

# Configure your Yubikey to use a static password.
# The password is set with -a and must be in lowercase hex format
sudo ykpersonalize -1 -ofixed=cccccccccccc -ayourhexkeyhereyourhexkeyhereyour -ostatic-ticket

# Setup up LUKS on a partition to be used as encrypted swap
sudo cryptsetup luksFormat --cipher aes-xts-plain64 --verify-passphrase --key-size 256 /dev/sdd3 

WARNING!
========
This will overwrite data on /dev/sdd3 irrevocably.

Are you sure? (Type uppercase yes): YES
Enter passphrase for /dev/sdd3: 
Verify passphrase: 
sudo cryptsetup open /dev/sdd3 SWAP
Enter passphrase for /dev/sdd3: 
sudo mkswap -L SWAP /dev/mapper/SWAP 
Setting up swapspace version 1, size = 15.3 GiB (16381898752 bytes)
LABEL=SWAP, UUID=0746c04e-7c96-4547-be07-9259d3218237


# Update /etc/crypttab
# <target name>  <source device>  <key file>  <options>
  SWAP           /dev/mapper/SWAP  none       luks

# Update /etc/initramfs-tools/conf.d/resume
RESUME=/dev/mapper/SWAP

# Install these (compcache package not found on my machine)
sudo apt-get install compcache dropbear busybox

# Add to /etc/initramfs-tools/initramfs.conf
BUSYBOX=y
DROPBEAR=y
COMPCACHE_SIZE=4G
COMPRESS=

# CompCache: https://lwn.net/Articles/334649/
# Another interesting sub-project is the SwapReplay infrastructure. This tool is meant to easily test memory allocator behavior under actual swapping conditions. It is a kernel module and a set of userspace tools to replay swap events in userspace. 

cd /etc/dropbear-initramfs/

ls -l
total 18
-rw-r--r-- 1 root root 556 2017-11-21 16:48:39 config
-rw------- 1 root root 458 2016-11-25 20:48:52 dropbear_dss_host_key
-rw------- 1 root root 242 2016-11-25 20:48:52 dropbear_ecdsa_host_key
-rw------- 1 root root 805 2016-11-25 20:48:52 dropbear_rsa_host_key

sudo /usr/lib/dropbear/dropbearconvert dropbear openssh dropbear_ecdsa_host_key id_ecdsa 
Key is a ecdsa-sha2-nistp521 key
Wrote key to 'id_ecdsa'

dropbearkey -y -f dropbear_ecdsa_host_key |grep "^ecdsa-sha2-nistp521 " | sudo tee id_ecdsa.pub
ecdsa-sha2-nistp521 AAAAE2VjZHNhLXNoYTItbmlzdHA1MjEAAAAIbmlzdHA1MjEAAACFBAG/k5HYPxmnbhEB085HW81kXnvMm3X0Oia7UpkGm285m1lMRoFkjdjlkGDH4qKs+cWOD1qftUOCfpSqi1+ksF/zWABWnt/OVdZ+OEXEdd4FWKRjzjZnOY2kmKz0F83AiX9h5uPwboJVOuaGLls/nt+Ex3rvyy9xVjplNOV6PMtuPpiBnw== root@workstation.jbsnet.xyz

cat ~/.ssh/authorized_keys | sudo tee authorized_keys
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJqkNAHN+bRtuGcPqGCA6Ahufu/UNboOil1hdSNEzZj3 jeremy+ed25519@jbsnet.xyz

echo " FUCK YOU, FBI ;-)" | boxes -d boy | sudo tee banner
         .-"""-.
        / .===. \
        \/ 6 6 \/
        ( \___/ )
   _ooo__\_____/______
  /                   \
 | FUCK YOU, FBI ;-)   |
  \_______________ooo_/
         |  |  |
         |_ | _|
         |  |  |
         |__|__|
         /-'Y'-\
        (__/ \__)



# Add to/replace in: /etc/default/dropbear
NO_START=0
DROPBEAR_BANNER="banner"
DROPBEAR_PORT=42

# Install /etc/initramfs-tools/hooks/crypt_unlock.sh
# See: https://gist.github.com/gusennan/712d6e81f5cf9489bd9f
# Inspect before executing any downloaded ecxecutable
sudo curl 'https://gist.githubusercontent.com/gusennan/712d6e81f5cf9489bd9f/raw/fda73649d904ee0437fe3842227ad8ac8ca487d1/crypt_unlock.sh' >  /etc/initramfs-tools/hooks/crypt_unlock.sh

sudo chmod +x /etc/initramfs-tools/hooks/crypt_unlock.sh

sudo zfs set compression=gzip bpool

# Add to:  /etc/dropbear-initramfs/config
DROPBEAR_OPTIONS="-p 42"

sudo update-initramfs -u
update-initramfs: Generating /boot/initrd.img-4.18.0-25-generic
cryptsetup: WARNING: could not determine root device from /etc/fstab
cryptsetup: WARNING: failed to detect canonical device of /dev/sdc1
dropbear: WARNING: Setting DROPBEAR in /etc/initramfs-tools/initramfs.conf is deprecated and will be ignored in a future release
dropbear: WARNING: Invalid authorized_keys file, remote unlocking of cryptroot via SSH won't work!

sudo update-grub
Sourcing file `/etc/default/grub'
Generating grub configuration file ...
Found background: /boot/grub/backgrounds/horsehead-nebula.png
Found background image: /boot/grub/backgrounds/horsehead-nebula.png
Found linux image: /boot/vmlinuz-4.18.0-25-generic
Found initrd image: /boot/initrd.img-4.18.0-25-generic
Found linux image: /boot/vmlinuz-4.15.0-76-generic
Found initrd image: /boot/initrd.img-4.15.0-76-generic
Found linux image: /boot/vmlinuz-4.15.0-70-generic
Found initrd image: /boot/initrd.img-4.15.0-70-generic
Found memtest86+ image: /BOOT/ubuntu/HEAD@/memtest86+.elf
Found memtest86+ image: /BOOT/ubuntu/HEAD@/memtest86+.bin
Found GRUB Invaders image: /boot/invaders.exec
device-mapper: reload ioctl on osprober-linux-sda4  failed: Device or resource busy
Command failed
device-mapper: reload ioctl on osprober-linux-sdb1  failed: Device or resource busy
Command failed
Found memdisk: /BOOT/ubuntu/HEAD@/memdisk
Found iso image: /boot/images/CorePlus-current.iso
Found floppy image: /boot/images/fdlinux-v300.img
Found floppy image: /boot/images/freedos.img
Found iso image: /boot/images/Samsung_SSD_830_CXM03B1Q.iso
Found iso image: /boot/images/Samsung_SSD_850_PRO_EXM04B6Q_Win.iso
done

# Try to reboot ...

# Try to hibernate ...

# Try to power on and resume from hibernation ...



 
```
