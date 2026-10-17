# PS3 Debian (Linux 6.7.0-rc2 & Built-in Jupiter Wi-Fi)

This repository provides kernel files, configuration scripts, and drivers to run modern Debian on the PlayStation 3 (tested on PS3 Slim with OtherOS++ / CFW Evilnat).

Respective files are stored under https://github.com/olin000/PS3-Debian

```text
Linux debian 6.7.0-rc2 #11 SMP ppc64 GNU/Linux
Debian GNU/Linux forky/sid (PowerPC 64-bit kernel, multi-arch userland)
```

---

## What's New: Working PS3 Built-in Wi-Fi (Jupiter) on Modern Linux!

Under Custom Firmware / OtherOS++, the Sony hypervisor disables the internal GELIC wireless VLAN, which is why official Sony OtherOS Wi-Fi never worked on custom firmware. 

Historically, the homebrew community accessed the internal Wi-Fi/Bluetooth composite hardware (`054c:036f` on internal USB Bus 3) using an out-of-tree driver written by Geoffrey Levand (`glevand`) called **Jupiter** (`ps3_jupiter` and `ps3_jupiter_sta`). However, `ps3_jupiter` was never merged into the upstream Linux kernel, and after Linux 3.12/3.16 it was abandoned and broke due to extensive Linux kernel API changes.

**The Jupiter Wi-Fi driver has now been updated and ported to Linux 6.7!**
- Hardware authentication with the internal Sony Eurus Wi-Fi chip is fully restored.
- Provides standard 802.11b/g station connectivity (`wlan0`) with WPA/WPA2-PSK support.
- Eliminates the need for external USB Wi-Fi dongles, freeing up the console's USB ports.
- Source code and pre-compiled 6.7.0-rc2 binaries are included under `wifi-jupiter/`.

---

## Repository Structure

- `boot/`: Pre-compiled kernel binaries (`vmlinux-6.7.0-rc2`, `initrd.img-6.7.0-rc2`, `config-6.7.0-rc2`, `System.map-6.7.0-rc2`), alongside legacy 6.3.0 files.
- `wifi-jupiter/`:
  - `src/`: Modernized source code (`Makefile`, `ps3_jupiter.c`, `ps3_jupiter_sta.c`, `ps3_jupiter.h`, `ps3_eurus.h`) compatible with modern 6.x kernels.
  - `bin/`: Pre-compiled 6.7.0-rc2 modules (`ps3_jupiter.ko`, `ps3_jupiter_sta.ko`).
- `etc/`: Configuration templates:
  - `modules`: Autoloads `ps3_jupiter`, `ps3_jupiter_sta`, and fallback `rt2800usb`.
  - `yaboot.conf`: Bootloader configuration for Petitboot / yaboot.
  - `NetworkManager/conf.d/10-unmanage-wlan0.conf`: Instructs NetworkManager to leave the WEXT interface alone.
  - `systemd/system/wpa_supplicant-wlan0.service`: Systemd service to associate the built-in Wi-Fi card on boot.
  - `systemd/network/20-wlan0.network`: Enables DHCP for `wlan0` via `systemd-networkd`.
  - `wpa_supplicant.conf.example`: Example Wi-Fi credentials configuration.
- `lib/modules/6.7.0-rc2/`: Full kernel modules tree for Linux 6.7.0-rc2.

---

## Notes & Prerequisites

1. **From-Scratch Setup Guide**: For a complete, step-by-step walkthrough covering target drive partitioning, QEMU-based debootstrap, Wi-Fi driver setup, and troubleshooting common pitfalls (such as Petitboot ext4 filesystem compatibility and mechanical drive spin-up delays), see the **[Complete Installation & Bootstrap Guide](INSTALL_GUIDE.md)**.
2. **Initial Installation (Alternative)**: The legacy installation method follows the guide on PSX-Place: [Installing Red Ribbon Linux on Rebug / Evilnat](https://www.psx-place.com/threads/tutorial-read-warning-installing-red-ribbon-linux-on-rebug-4-81-2-4-84-2-d-rex.16419/).
3. **Firmware**: Tested on Cobra CFW (e.g. CFW Evilnat 4.9x with OtherOS++ support).
4. **Bootloader Config**: Keep `yaboot.conf` in `/etc/` of your Linux root partition so Petitboot can detect boot options automatically.
5. **USB Wi-Fi Fallback**: If you prefer or need to use an external USB Wi-Fi dongle (such as a Ralink RT5572 with the `rt2800usb` driver), the `rt2800usb` module is preserved in `/etc/modules` and the 6.7.0-rc2 module tree. NetworkManager will automatically manage the USB stick (`wlu1`) if plugged in.

---

## How to Enable the Built-in Wi-Fi (Jupiter)

### 1. Install the Kernel Modules
Copy the pre-compiled modules to your kernel modules directory:
```bash
sudo mkdir -p /lib/modules/6.7.0-rc2/kernel/drivers/net/wireless/ps3jupiter
sudo cp wifi-jupiter/bin/ps3_jupiter*.ko /lib/modules/6.7.0-rc2/kernel/drivers/net/wireless/ps3jupiter/
sudo depmod -a 6.7.0-rc2
```
*(Or compile them from source in `wifi-jupiter/src` by running `make`)*.

### 2. Autoload at Boot
Add `ps3_jupiter` and `ps3_jupiter_sta` to `/etc/modules`:
```bash
sudo tee -a /etc/modules << EOF
ps3_jupiter
ps3_jupiter_sta
EOF
```

### 3. Configure Wi-Fi Credentials
Create `/etc/wpa_supplicant.conf`:
```text
ctrl_interface=/run/wpa_supplicant
update_config=1

network={
    ssid="YOUR_WIFI_SSID"
    key_mgmt=WPA-PSK
    psk="YOUR_WIFI_PASSWORD"
}
```

### 4. Enable Services
Copy the provided service and network configs:
```bash
# Prevent NetworkManager from mistaking wlan0 for a wired port
sudo cp etc/NetworkManager/conf.d/10-unmanage-wlan0.conf /etc/NetworkManager/conf.d/

# Enable WPA supplicant and DHCP on wlan0
sudo cp etc/systemd/system/wpa_supplicant-wlan0.service /etc/systemd/system/
sudo cp etc/systemd/network/20-wlan0.network /etc/systemd/network/

sudo systemctl daemon-reload
sudo systemctl enable wpa_supplicant-wlan0.service systemd-networkd.service
```

### 5. Reboot and Disconnect USB Wi-Fi Stick
Reboot the console (`sudo reboot`). On reboot, `ps3_jupiter` initializes the internal Eurus hardware, connects to your Wi-Fi network, and obtains an IP address automatically over DHCP.

---

Hope this helps the PS3 Linux community! :)