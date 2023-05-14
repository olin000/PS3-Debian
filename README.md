# PS3 Debian

As a proof of concept, I have managed to successfully upgrade PS3 to Linux debian 6.3.0 kernel on my PS3 Slim.

Respective files are stored under https://github.com/olin000/PS3-Debian

Linux debian 6.3.0 #4 SMP Sat May 13 21:11:36 CEST 2023 ppc64 GNU/Linux

Distributor ID: Debian
Description:    Debian GNU/Linux 12 (bookworm)
Release:    12
Codename:   bookworm

Notes:
1. The initial steps are as described in the thread https://www.psx-place.com/threads/tutorial-read-warning-installing-red-ribbon-linux-on-rebug-4-81-2-4-84-2-d-rex.16419/ with this main difference that I was not partitioning the console's hard drive, but rather installing Red Ribbon GNU/Linux for PS3 from one USB memory stick (where the ISO got unpacked) to another USB memory stick
2. At the time of writing this I am using Cobra version 8.4 Cobra fw version 4.9 (CFW Evilnat)
3. For your convenience it is important to have yaboot.conf script stored in /etc/ of the Linux's (not Petitboot's) drive. It went missing for me in one of the steps (probably during the packages update). Otherwise one might be forced to boot Linux using kexec command (see \etc\petit_script)
4. I could not make use of the old modules/drivers for the PS3 wireless card (Jupiter), not to say about finding their new version. Hence I am using my WiFI dongle Ralink Technology, Corp. RT5572 Wireless Adapter. The kernel module for it is rt2800usb. You might need to additionally install one of the "firmware-*" packages for you card to work like this.

Steps:
1. Install Red Ribbon GNU/Linux for PS3
2. Update the apt sources to "deb http://ftp.ports.debian.org/debian-ports/ sid main"
3. Upgrade the packages. This I see as most tricky. There were many cross/missing dependencies in the packages and often the installation had to be forced. At the end when it was all fixed, I forced reinstalled all the packages, just to be sure
4. Install additional packages. E.g. git
5. Clone the current Linux kernel with "git clone https://kernel.googlesource.com/pub/scm/linux/kernel/git/geoff/ps3-linux"
6. Make and install the new kernel the usual way (compiled version of the kernel is stored in my GitHub). I used the default config, just adding the rt2800usb (and in general wireless) modules to it.

Hope this helps :)
