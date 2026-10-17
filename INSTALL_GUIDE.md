# PS3 Debian: Complete Installation & Bootstrap Guide

This guide walks you through bootstrapping a fresh, modern **Debian Sid (PowerPC)** installation from scratch onto a dedicated USB drive or external SSD, with **Linux 6.7.0-rc2**, **built-in Jupiter Wi-Fi**, and **multi-tier swap**, ready to boot on a PlayStation 3 running Custom Firmware (CFW Evilnat / OtherOS++).

---

## Table of Contents
1. [Prerequisites](#1-prerequisites)
2. [Step 1: Partitioning the Target USB Drive (MBR)](#step-1-partitioning-the-target-usb-drive-mbr)
3. [Step 2: Formatting Filesystems (Critical Petitboot Compatibility)](#step-2-formatting-filesystems-critical-petitboot-compatibility)
4. [Step 3: Cross-Architecture Debian Bootstrap](#step-3-cross-architecture-debian-bootstrap)
5. [Step 4: APT Repositories & Essential Packages](#step-4-apt-repositories--essential-packages)
6. [Step 5: User Setup & Permissions](#step-5-user-setup--permissions)
7. [Step 6: Deploying Kernel, Modules & Built-in Wi-Fi](#step-6-deploying-kernel-modules--built-in-wi-fi)
8. [Step 7: Configuring Bootloader & RSX VRAM Swap](#step-7-configuring-bootloader--rsx-vram-swap)
9. [Step 8: Safely Unmounting & First Boot](#step-8-safely-unmounting--first-boot)
10. [Troubleshooting](#troubleshooting)

---

## 1. Prerequisites

- **PlayStation 3**: Slim or Fat console running Custom Firmware with OtherOS++ support (tested on Evilnat 4.9x).
- **Storage**: A dedicated USB 3.0 flash drive, external USB HDD, or USB SSD (16 GB to 1 TB+).
- **Host Workstation**: A PC or laptop running Linux (Ubuntu, Debian, Fedora, or WSL2 on Windows) to prepare the USB drive.
- **This Repository**: Cloned on your host workstation:
  ```bash
  git clone https://github.com/olin000/PS3-Debian.git
  cd PS3-Debian
  ```

---

## Step 1: Partitioning the Target USB Drive (MBR)

> [!IMPORTANT]
> The PS3's Petitboot bootloader **requires a classic DOS / MBR partition table**. Do not use GPT!

1. Plug your target USB drive into your host Linux PC and identify its device node:
   ```bash
   lsblk
   ```
   *(Identify your drive carefully, e.g. `/dev/sdX`. **Do not overwrite your PC's internal drive!**)*

2. Open `fdisk`:
   ```bash
   sudo fdisk /dev/sdX
   ```
3. Create the partitions:
   - Type `o` to create a new empty DOS/MBR partition table.
   - **Root Partition (Partition 1)**:
     - Type `n`, then `p`, then `1`, press `Enter` (default start sector `2048`).
     - At "Last sector", type `-2G` and press `Enter` *(allocates the disk to root while reserving exactly 2 GB at the end for swap)*.
   - **Swap Partition (Partition 2)**:
     - Type `n`, then `p`, then `2`, press `Enter`, press `Enter` *(uses the remaining 2 GB)*.
     - Type `t`, then `2`, then `82` *(sets type to Linux swap)*.
   - Type `w` to write the table to disk.

---

## Step 2: Formatting Filesystems (Critical Petitboot Compatibility)

> [!WARNING]
> **Modern `ext4` Compatibility Trap**: Modern Linux distributions (Ubuntu 24+, Debian 12+) enable features like `orphan_file` and `metadata_csum_seed` by default in `mkfs.ext4`. Petitboot's built-in kernel cannot parse these newer features and will refuse to mount the drive (`unrecognised features on journal`).

Format the partitions with backwards-compatible flags:

```bash
# 1. Format Root partition (ext4 without unsupported modern journal flags)
sudo mkfs.ext4 -b 4096 -O ^orphan_file,^metadata_csum_seed -L ps3root /dev/sdX1

# (Optional recommended) Run without journal for zero Petitboot journal errors and reduced USB wear:
sudo tune2fs -O ^has_journal /dev/sdX1

# 2. Format Swap partition
sudo mkswap -L ps3swap /dev/sdX2

# 3. Print and record your partition UUIDs:
sudo blkid /dev/sdX1 /dev/sdX2
```
*Take note of the UUIDs for `/dev/sdX1` and `/dev/sdX2`—you will need them in `/etc/fstab` and `/etc/yaboot.conf`.*

---

## Step 3: Cross-Architecture Debian Bootstrap

Mount the newly formatted root partition:
```bash
sudo mkdir -p /mnt/usb
sudo mount /dev/sdX1 /mnt/usb
```

### 1. Enable PowerPC Emulation on x86 Host
Because your PC runs x86_64 and the PS3 runs PowerPC (32-bit `powerpc` userland), install QEMU user-mode binfmt support:
```bash
sudo apt update
sudo apt install -y qemu-user-binfmt binfmt-support debootstrap
```
Verify the kernel has registered the `qemu-ppc` interpreter with the `F` (Fix-binary) flag:
```bash
cat /proc/sys/fs/binfmt_misc/qemu-ppc | grep flags
# Expected output: flags: POF (or similar containing F)
```

### 2. Bootstrap Debian Ports (Sid)
Run `debootstrap` to install the minimal base system:
```bash
sudo debootstrap --arch=powerpc --no-check-gpg sid /mnt/usb http://deb.debian.org/debian-ports
```
*(This downloads, unpacks, and configures the base Debian PowerPC system directly onto your USB drive).*

---

## Step 4: APT Repositories & Essential Packages

### 1. Install the Official Debian Ports GPG Keyring
Download and install the official `debian-ports-archive-keyring` into the chroot:
```bash
sudo curl -sL "http://deb.debian.org/debian-ports/pool/main/d/debian-ports-archive-keyring/debian-ports-archive-keyring_2026.02.04_all.deb" -o /mnt/usb/root/keyring.deb
sudo chroot /mnt/usb dpkg -i /root/keyring.deb
sudo rm -f /mnt/usb/root/keyring.deb
```

### 2. Configure Modernized APT Sources (`deb822`)
Copy the modernized repository source from this repo into the chroot:
```bash
sudo mkdir -p /mnt/usb/etc/apt/sources.list.d
sudo cp etc/apt/sources.list.d/debian-ports.sources /mnt/usb/etc/apt/sources.list.d/
sudo rm -f /mnt/usb/etc/apt/sources.list
```

### 3. Install Essential Packages
A bare debootstrap does not include networking, SSH, or sudo. Install them now:
```bash
sudo chroot /mnt/usb apt update
sudo chroot /mnt/usb apt install -y wpasupplicant openssh-server sudo wireless-tools net-tools nano
sudo chroot /mnt/usb systemctl enable ssh
```

---

## Step 5: User Setup & Permissions

> [!TIP]
> Inside QEMU chroot, modern `useradd -m` may report `Bad file descriptor` due to `copy_file_range` syscall emulation. Create the user directory manually as shown below:

```bash
# Set root password
echo "root:ps3" | sudo chroot /mnt/usb chpasswd

# Create user 'ps3' and create home directory cleanly
sudo chroot /mnt/usb useradd -s /bin/bash ps3
sudo mkdir -p /mnt/usb/home/ps3
sudo cp /mnt/usb/etc/skel/.bashrc /mnt/usb/home/ps3/ 2>/dev/null || true
sudo cp /mnt/usb/etc/skel/.profile /mnt/usb/home/ps3/ 2>/dev/null || true
sudo chown -R 1000:1000 /mnt/usb/home/ps3
echo "ps3:ps3" | sudo chroot /mnt/usb chpasswd

# Grant sudo permissions
sudo chroot /mnt/usb usermod -aG sudo ps3
echo "ps3 ALL=(ALL) NOPASSWD: ALL" | sudo tee /mnt/usb/etc/sudoers.d/ps3 > /dev/null
sudo chmod 0440 /mnt/usb/etc/sudoers.d/ps3
```

Set hostname:
```bash
echo "ps3-debian" | sudo tee /mnt/usb/etc/hostname
sudo bash -c 'cat << "EOF" > /mnt/usb/etc/hosts
127.0.0.1   localhost
127.0.1.1   ps3-debian
EOF'
```

---

## Step 6: Deploying Kernel, Modules & Built-in Wi-Fi

From the root of this repository:

### 1. Copy Kernel & Boot Artifacts
```bash
sudo cp boot/vmlinux-6.7.0-rc2 /mnt/usb/boot/
sudo cp boot/initrd.img-6.7.0-rc2 /mnt/usb/boot/
```

### 2. Copy Kernel Modules & Jupiter Wi-Fi Driver
```bash
sudo cp -r lib/modules/6.7.0-rc2 /mnt/usb/lib/modules/
sudo mkdir -p /mnt/usb/lib/modules/6.7.0-rc2/extra
sudo cp wifi-jupiter/bin/ps3_jupiter*.ko /mnt/usb/lib/modules/6.7.0-rc2/extra/

# Rebuild module dependency map
sudo depmod -a -b /mnt/usb 6.7.0-rc2
```

### 3. Autoload Wi-Fi on Boot
```bash
sudo cp etc/modules /mnt/usb/etc/modules
sudo cp etc/modprobe.d/jupiter.conf /mnt/usb/etc/modprobe.d/jupiter.conf
```

### 4. Configure Wi-Fi Credentials & Services
```bash
# Configure Wi-Fi network and credentials
sudo bash -c 'cat << "EOF" > /mnt/usb/etc/wpa_supplicant.conf
ctrl_interface=/run/wpa_supplicant
update_config=1

network={
    ssid="YOUR_WIFI_SSID"
    psk="YOUR_WIFI_PASSWORD"
    key_mgmt=WPA-PSK
}
EOF'
sudo chmod 600 /mnt/usb/etc/wpa_supplicant.conf

# Deploy service and network configs
sudo cp etc/systemd/system/wpa_supplicant-wlan0.service /mnt/usb/etc/systemd/system/
sudo cp etc/systemd/network/20-wlan0.network /mnt/usb/etc/systemd/network/
sudo cp etc/NetworkManager/conf.d/10-unmanage-wlan0.conf /mnt/usb/etc/NetworkManager/conf.d/ 2>/dev/null || true
sudo cp etc/NetworkManager/conf.d/no-dns.conf /mnt/usb/etc/NetworkManager/conf.d/ 2>/dev/null || true

# Static DNS fallback
sudo bash -c 'cat << "EOF" > /mnt/usb/etc/resolv.conf
nameserver 1.1.1.1
nameserver 8.8.8.8
EOF'

# Enable networking services
sudo chroot /mnt/usb systemctl enable systemd-networkd.service
sudo chroot /mnt/usb systemctl enable wpa_supplicant-wlan0.service
```

---

## Step 7: Configuring Bootloader & RSX VRAM Swap

### 1. Configure `/etc/fstab`
Edit `/mnt/usb/etc/fstab` and insert your actual partition UUIDs:
```bash
sudo bash -c 'cat << "EOF" > /mnt/usb/etc/fstab
# /etc/fstab: static file system information.
proc                                        /proc           proc    defaults                              0 0
UUID=PUT_YOUR_ROOT_UUID_HERE               /               ext4    relatime,defaults,errors=remount-ro   0 1
UUID=PUT_YOUR_SWAP_UUID_HERE               none            swap    sw,pri=10                             0 0
EOF'
```

### 2. Configure Petitboot (`/etc/yaboot.conf`)
Copy and configure `yaboot.conf`:
```bash
sudo cp etc/yaboot.conf /mnt/usb/etc/yaboot.conf
sudo cp etc/yaboot.conf /mnt/usb/etc/kboot.conf
```
Ensure your `6.7.0-rc2` entry in `/mnt/usb/etc/yaboot.conf` has your Root UUID and `rootdelay=10`:
```text
image=/boot/vmlinux-6.7.0-rc2
        label=6.7.0-rc2
        initrd=/boot/initrd.img-6.7.0-rc2
        root=UUID=PUT_YOUR_ROOT_UUID_HERE
        append="video=ps3fb nomodeset nosplash rootwait rootdelay=10"
```
*(Keep `rootdelay=10` especially if using an external mechanical USB hard drive, allowing the disk motor enough time to spin up before root detection).*

### 3. Deploy High-Priority RSX VRAM Swap (256 MB Fast Swap)
```bash
sudo cp etc/rc.local /mnt/usb/etc/rc.local
sudo chmod +x /mnt/usb/etc/rc.local
sudo cp etc/systemd/system/rc-local.service /mnt/usb/etc/systemd/system/
sudo chroot /mnt/usb systemctl enable rc-local.service
```

---

## Step 8: Safely Unmounting & First Boot

> [!CAUTION]
> Mechanical USB drives have an internal DRAM hardware write cache. Always flush and power down properly before physically disconnecting the drive from your host PC.

```bash
cd ~
sync
sudo umount /mnt/usb
sudo udisksctl power-off -b /dev/sdX 2>/dev/null || true
```

1. Plug your USB drive into **USB Port 1** on the PlayStation 3 (the right-most USB port).
2. Power on the console $\rightarrow$ Petitboot launches.
3. Select `6.7.0-rc2` (or wait for autoboot).
4. System boots to the login prompt!
   - **User**: `ps3`
   - **Password**: `ps3`
5. Connect over SSH / PuTTY:
   ```bash
   ssh ps3@<PS3-IP-ADDRESS>
   ```

---

## Troubleshooting

### 1. Petitboot reports `unrecognised features on journal`
Modern ext4 journal features were added by your host PC. Reconnect the drive to your host and remove the journal:
```bash
sudo tune2fs -O ^has_journal /dev/sdX1
sudo e2fsck -f -y /dev/sdX1
```

### 2. Drops to BusyBox / `scripts/local-block` fails
The mechanical drive did not spin up in time before initramfs searched for the UUID.
- From BusyBox: mount manually and continue booting:
  ```sh
  mount /dev/sda1 /root
  exit
  ```
- To fix permanently: Ensure `rootdelay=10` is present in `/etc/yaboot.conf`.

### 3. `usb 3-2: no free Tx URBs` in dmesg
This is a harmless Wi-Fi queue notification during packet bursts. Ensure `options ps3_jupiter_sta max_txurbs=64` is placed in `/etc/modprobe.d/jupiter.conf` to provide 4x buffer headroom.

### 4. Apt reports `Missing key 5FFACC4B97F15BF3...`
Install the official Debian Ports keyring:
```bash
curl -sL "http://deb.debian.org/debian-ports/pool/main/d/debian-ports-archive-keyring/debian-ports-archive-keyring_2026.02.04_all.deb" -o keyring.deb
sudo dpkg -i keyring.deb
```