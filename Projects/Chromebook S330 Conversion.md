**Device Info:**
- Lenovo Chromebook S330
- MediaTek MT8173C (ARM64) CPU
- Oak Family

#### Original Goal W/ Chromebook S330:
- I wanted to wipe ChromeOS and install a Linux Distro to have another endpoint for a Wazuh-Agent, specifically an endpoint not tied to a virtual machine and one that's suitable for attempted exploitation and hardening

#### Some of the Obstacles I Faced:
1. ARM Chromebooks cannot:
    1. Replace firmware with specially made tools like the MrChromeBox UEFI
    2. Boot standard UEFI Linux installers
    3. Erase ChromeOS at firmware level
    4. essentially, **ChromeOs cannot be fully wiped on ARM Devices**
2. Standard Linux Image Files Don't Boot on Arm (aarch64)
3. Crouton not working when fully in Developer Mode
4. Trying to use sudo commands in crosh terminal (> sudo commands will not succeed by default)
5. ChromeOS being massively restricted

#### Technical Steps to Complete Goal:
1. Entered Developer Mode waiting specifically for screen on reboot to say "OS Not Verified", then pressing Ctrl + D to boot
2. Enabled USB Boot via CLI in the VT-2 terminal **On My Desktop PC:**
3. Downloaded the correct image for my Chromebook device family from a GitHub Repo from hexdump0815 ([https://github.com/hexdump0815/linux-mainline-on-arm-chromebooks](https://github.com/hexdump0815/linux-mainline-on-arm-chromebooks)) 
	**Follow These Links to Navigate Through Repo Like I Did:** 3.1) [https://github.com/hexdump0815/imagebuilder/blob/main/systems/chromebook_oak/readme.md](https://github.com/hexdump0815/imagebuilder/blob/main/systems/chromebook_oak/readme.md) 3.2) [https://github.com/velvet-os/imagebuilder/releases/tag/251115-01](https://github.com/velvet-os/imagebuilder/releases/tag/251115-01) 3.3) Download Image File: [https://github.com/velvet-os/imagebuilder/releases/download/251115-01/chromebook_corsola-aarch64-trixie-251015.img.gz](https://github.com/velvet-os/imagebuilder/releases/download/251115-01/chromebook_corsola-aarch64-trixie-251015.img.gz)
4. Extracted the contents using 7-zip and flashed them to a USB using Rufus and safely unmounted after flashing
5. Turned off Chromebook, then plugged in USB and pressed Ctrl + U
6. Then logged into Velvet-OS Debian with the default credentials
7. Immediately opened the terminal and did sudo apt update and sudo apt full-upgrade -y

### End Result:
1. Velvet-Os Debian is running smoothly
2. Wazuh-Agent installed on endpoint
3. No more ChromeOs Limitations
4. Laptop is faster because Velvet-Os is light