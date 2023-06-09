
# How to build your own Distro

## Compile Tina

Please refer to the repository for the CrealityOS: https://github.com/CrealityTech/sonic_pad_os

Follow their instructions to compile it, additionally you can try to flash it just to see if it works.

### About the procedure:

Using mainline's kernel is out of the question since it requires serious patching because of the lack of support of the SOC (R818). 

To boot Debian, we are going to use the kernel provided by AllWinner and replace the rootfs before packing it.

Originally, Tina uses a Read Only filesystem (squashfs) with an OverlayFS, I assume this is done to avoid corruption. 

We are going to replace this behaviour by using a single EXT4 partition.

## Configure Tina with an EXT4 rootfs:
The steps are as follow: 

cd into the "sonic_pad_os" repo:
```
cd **replace with the sonic_pad_os dir**
```

Config the Tina enviroment:
```
source build/envsetup.sh
```

"Lunch":
```
lunch 6
```

Configure EXT4:
```
make menuconfig
```

A menu will appear:
![makemenu](https://github.com/Jpe230/SonicPad-Debian/assets/6202305/f0b3d392-0dbd-4cf9-bc78-2114c3b07d03)

Go to `Target Images`:
![TargetImages](https://github.com/Jpe230/SonicPad-Debian/assets/6202305/38acb4f0-04fe-4c0f-ae20-1823559f02d5)

Disable `squashfs` and enable `ext4` (you will need to setup ext4 partition):
![ext4 conf](https://github.com/Jpe230/SonicPad-Debian/assets/6202305/e1f502e0-a58d-4664-b95b-9c59700956f9)

Make the ext4 partition bigger, I use 1GB.
![fs-size](https://github.com/Jpe230/SonicPad-Debian/assets/6202305/7a2146dc-5d02-4c57-a89e-ade835c32d07)

You can always increase the size if your rootfs doesn't fit.  

Edit the following file:
```
device/config/chips/r818/configs/sonic_lcd/linux/sys_partition.fex
```

Remove the partitions after `rootfs` and increase the size for the `rootfs` partition, example:
```
...
[partition]
    name         = rootfs
    size         = 14542431
    downloadfile = "rootfs.fex"
    user_type    = 0x8000
```
The size is in blocks, in the example I set 7GB for the rootfs

Compile Tina with: 

```
make
```
  
Make sure that it compiled, and a `rootfs.img` file was generate at `sonic_pad_os/out/r818-sonic_lcd/`

Note: you can reduce the compilation time by disabling Creality's packages and modifying the Makefile. See [this commit](https://github.com/Jpe230/SonicPadOS/commit/3c7059261dbf0e7579e717690b7f6a8fd12529a0).



## Create a Debian RootFS

We are going to use `debootstrap` to create a rootfs, but you are free to choose whatever distro you want.

Run the following:
```
# Run as sudo
sudo -i

# Install necessary packages:
sudo apt-get install qemu-user-static debootstrap

# Use `debootstrap` to create a basic rootfs:
debootstrap --no-check-gpg --foreign --verbose --arch=armhf buster rootfs http://deb.debian.org/debian/

# Copy qemu into the rootfs:
cp /usr/bin/qemu-arm-static rootfs/usr/bin/
chmod +x rootfs/usr/bin/qemu-arm-static

# Chroot into the filesystem:
LC_ALL=C LANGUAGE=C LANG=C chroot rootfs
```

The following is meant to be run inside the `chrooted rootfs`:
```
# Run debootstrap:
/debootstrap/debootstrap --second-stage --verbose

# Set hostname
echo "SonicPad" > /etc/hostname

# Set hostname in hosts file
echo "127.0.1.1 SonicPad" >> /etc/hosts

# Configure root password:
passwd

# Update sources
apt update

# Install utils packages
apt install git net-tools wpasupplicant build-essential locales openssh-server wget libssl-dev sudo

# Install cmake for klipperscreen:
cd
wget https://github.com/Kitware/CMake/releases/download/v3.26.4/cmake-3.26.4.tar.gz
tar -xvzf cmake-3.26.4.tar.gz
rm cmake-3.26.4.tar.gz
cd cmake-3.26.4
./bootstrap
make
make install
cd ../
rm -rf cmake-3.26.4

# Configure fstab:
echo "PARTLABEL="rootfs" / ext4 noatime,lazytime,rw 0 0" > /etc/fstab

# Configure rc local
ln -fs /lib/systemd/system/rc-local.service /etc/systemd/system/rc-local.service
touch /etc/rc.local
chmod +x /etc/rc.local
systemctl enable rc.local

# Configure WiFi MACADDRESS (Generate your own)
mkdir /etc/wifi
echo "FC:EE:92:11:14:05" > /etc/wifi/xr_wifi.conf

# Clean cache
apt clean
rm -rf /var/cache/apt/

# Configure user
adduser <your username>
usermod -aG sudo <your username>

# Exit roofs
exit
```

Copy the kernel modules located at `sonic_pad_os/out/r818-sonic_lcd/staging_dir/target/rootfs/lib` to the rootfs

```
cp -r /home/[path to repo]/sonic_pad_os/out/r818-sonic_lcd/staging_dir/target/rootfs/lib/firmware/ rootfs/lib/
cp -r /home/[path to repo]/sonic_pad_os/out/r818-sonic_lcd/staging_dir/target/rootfs/lib/modules/ rootfs/lib/
```

Edit `rootfs/etc/rc.local`:

You will need to load GPU, Wifi, configure dhclient, you can find an example at this repo: `scripts/rc.local`

Create `rootfs/etc/wpa_supplicant.conf`:

Generate a valid conf, you can find an example at this repo: `scripts/wpa_supplicant.conf`


Dont forget to exit the root account
```
exit
```

## Replace Rootfs with ours

Assuming you have compiled Tina:

Cd to `out` folder
```
cd sonic_pad_os/out/r818-sonic_lcd/
```

Mount Tina `rootfs` 

```
mkdir mountfs && sudo mount -o loop rootfs.img mountfs
```

Delete Tina `rootfs` contents
```
cd mountfs
sudo rm -rf *
```
Copy our rootfs to the mount dir
```
sudo -i
cp -rfp rootfs/* /home/[path to repo]/sonic_pad_os/out/r818-sonic_lcd/mountfs/
exit
```

Unmount image
```
sudo umount mountfs
```
Repair image, accept all prompts
```
tune2fs -O^resize_inode rootfs.img
e2fsck -f rootfs.img
fsck.ext4 rootfs.img
```

Pack the update image:

```
pack
```

Flash with Livesuit or PhoenixSuit

## After flash

1) Get the IP of the Pad with help of your router, or by any other means
2) SSH Into
3) Once logged in, resize the fs using: (double check that the partition is correct!!!!)
```
sudo resize2fs /dev/mmcblk0p7
```
4) Verify disk space with:
```
df -h
```

## Install Klipper, Mainsail and KlipperScreen

Please refer to the installation instructions of KIAUH: https://github.com/th33xitus/kiauh

**After installing KlipperScreen, don't shutdown or reboot the pad.**

Create a config for X11:
```
sudo nano /etc/X11/xorg.conf
```

And add the following:
```
Section "Device"
        Identifier      "Allwinner R818 FBDEV"
        Driver          "fbdev"
        Option          "fbdev" "/dev/fb0"

        Option          "SwapbuffersWait" "true"
EndSection
```

Finally, reboot the pad.