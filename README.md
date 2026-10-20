# Full Native Xubuntu Installation on a USB Drive (MBR/Legacy)

Have you ever wanted a full-fledged Linux system running directly on a USB flash drive? If so, this guide is for you! 

Unlike traditional live-USB persistence (which uses a compressed loopback overlay file), this method performs a **true native installation** using `debootstrap`. Your USB drive is treated just like an internal hard drive—every file, package, and setting is saved natively.

> **Notice:** I am not a part of the Xubuntu team or their affiliates, so please give them the primary credit for their amazing desktop environment.

---

## Requirements
* **Target PC/Laptop:** Legacy BIOS / MBR system.
* **RAM:** At least 2GB of RAM.
* **USB Drive:** At least 16GB of storage.

---

## Step-by-Step Installation

### STEP 1: IDENTIFY YOUR USB DRIVE
Plug in your USB drive and run `lsblk` to find its device name:
```bash
lsblk
```
Example output:
```text
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda      8:0    0 298.1G  0 disk 
├─sda1   8:1    0    10G  0 part [SWAP]
└─sda2   8:2    0 288.1G  0 part /
sdb      8:16   1  28.6G  0 disk 
├─sdb1   8:17   1  28.6G  0 part 
└─sdb2   8:18   1    32M  0 part
```
> *Note: Ignore the loop devices. If your computer uses an internal HDD as `sda`, your USB drive will likely be `sdb`. Adjust `/dev/sdb` accordingly for your system to avoid overwriting your main OS!*

---

### STEP 2: WIPE AND PARTITION THE DRIVE
Wipe any existing partition tables on the USB drive:
```bash
sudo wipefs -a /dev/sdb
```
Open `cfdisk` to create a new MBR layout:
```bash
sudo cfdisk /dev/sdb
```
1. Select **dos** as the label type.
2. Select **Free space**, choose **New**, and allocate space leaving ~1-2GB for swap.
3. Choose **Primary**, and make sure it is marked **bootable** (an asterisk `*` should appear on `sdb1`).
4. Select the remaining free space, choose **New**, **Primary**, and set its type to **Linux swap / Solaris (82)**.
5. Select **Write**, type `yes` to confirm, and then choose **Quit**.

---

### STEP 3: FORMAT AND MOUNT PARTITIONS
Format your root partition as ext4, set up swap, and mount everything to `/mnt`:
```bash
sudo mkfs.ext4 -F -L "xubuntu-usb" /dev/sdb1
sudo mkswap /dev/sdb2
sudo swapon /dev/sdb2
sudo mkdir -p /mnt
sudo mount /dev/sdb1 /mnt
```

---

### STEP 4: BOOTSTRAP THE UBUNTU BASE SYSTEM
Install bootstrapping tools, clear out any stale mount files, and run `debootstrap` for Ubuntu 24.04 (`noble`):
```bash
sudo apt update
sudo apt install debootstrap arch-install-scripts
sudo rm -rf /mnt/*

sudo debootstrap \
  --arch amd64 \
  --components=main,restricted,universe,multiverse \
  --include=linux-image-generic,linux-headers-generic,linux-firmware,network-manager,wget,sudo,nano,locales,grub-pc,xubuntu-core,lightdm \
  noble \
  /mnt \
  https://archive.ubuntu.com/ubuntu
```

Generate the `fstab` file automatically using `genfstab`:
```bash
sudo genfstab -P /mnt | sudo tee /mnt/etc/fstab
```

#### TROUBLESHOOTING: "E: Couldn't download packages" Error
If `debootstrap` fails with a list of un-downloaded packages (due to minor network timeouts or archive hiccups), do NOT wipe `/mnt`. Simply re-run the exact same debootstrap command:
```bash
sudo debootstrap \
  --arch amd64 \
  --components=main,restricted,universe,multiverse \
  --include=linux-image-generic,linux-headers-generic,linux-firmware,network-manager,wget,sudo,nano,locales,grub-pc,xubuntu-core,lightdm \
  noble \
  /mnt \
  https://archive.ubuntu.com/ubuntu
```
`debootstrap` will reuse already downloaded packages and fetch missing ones. Alternatively, once inside the chroot environment (Step 5), run: 
```bash
apt update && apt --fix-broken install -y
```

---

### STEP 5: CONFIGURE INSIDE THE CHROOT ENVIRONMENT
Jump into your new system using `arch-chroot`:
```bash
sudo arch-chroot /mnt
```

#### 1. Set up Hostname
Set the system hostname to `xubuntu`:
```bash
echo "xubuntu" > /etc/hostname
cat << 'EOF' > /etc/hosts
127.0.0.1 localhost
127.0.1.1 xubuntu

# The following lines are desirable for IPv6 capable hosts
::1 localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
EOF
```

#### 2. Set up APT Sources
Clear out the default list and write the modern DEB822 sources list:
```bash
> /etc/apt/sources.list
cat << 'EOF' > /etc/apt/sources.list.d/ubuntu.sources
Types: deb
URIs: http://archive.ubuntu.com/ubuntu/
Suites: noble noble-updates noble-backports
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

Types: deb
URIs: http://security.ubuntu.com/ubuntu/
Suites: noble-security
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
EOF

apt update
apt --fix-broken install -y
apt upgrade -y
```

#### 3. Create User Accounts & Passwords
```bash
useradd -m -s /bin/bash user
passwd user
passwd root
usermod -aG sudo,plugdev,audio,video,storage user
```

#### 4. Configure Locale and Timezone
```bash
dpkg-reconfigure locales
```
*(Select your preferred system language when prompted).*

For your timezone (e.g., Asia/Kolkata):
```bash
ln -sf /usr/share/zoneinfo/Asia/Kolkata /etc/localtime
echo "Asia/Kolkata" > /etc/timezone
```

#### 5. Install Extra Desktop Utilities & Browser
```bash
apt install -y firefox xfce4-goodies dbus-x11 pciutils
```

#### 6. Install and Configure GRUB for MBR
Install GRUB directly onto the root disk (`/dev/sdb`, **NOT** the partition `sdb1`):
```bash
grub-install /dev/sdb
grub-mkconfig -o /boot/grub/grub.cfg
```

#### 7. Setup Network Interfaces Config
Ensure the loopback interface is defined:
```bash
mkdir -p /etc/network
cat << 'EOF' > /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).
source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback
EOF
sync
```

---

### STEP 6: CLEAN UP AND EXIT
Exit the chroot environment and unmount the USB drive safely:
```bash
exit
sudo swapoff /dev/sdb2
sudo umount /mnt
```

---

## DONE!
Reboot your PC, enter your BIOS boot menu, and select your USB drive. You now have a fully functioning, native Xubuntu installation with complete persistence!