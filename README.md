# OpenWrt on Linksys MR9000 — Serial-Free Installation Guide & Technical Discoveries

> **⚠️ DISCLAIMER**
> This document is a personal record of experience and research, not an official guide or documentation. Following these steps is entirely at your own risk. Flashing firmware always carries a risk of bricking your device. The author takes no responsibility for any damage, data loss, or bricked devices resulting from following this guide. The Linksys MR9000 is a relatively expensive modern router — proceed with caution and make sure you understand each step before performing it.

> **📝 ACCURACY NOTE**
> If you find any technical details in this guide that are inaccurate, incomplete, or incorrect, please open an issue or submit a pull request to help improve it. This guide is based on personal experience and investigation — corrections are always welcome.

---

## Overview

The Linksys MR9000 is a powerful tri-band router that is not yet officially supported by OpenWrt. This guide documents a **serial-free installation method** — meaning you can install OpenWrt without opening the router, without attaching any hardware tools, and with the rubber feet perfectly intact. 😄

This guide also documents several technical discoveries made during the installation process that may be useful to the community and could help push the MR9000 toward official OpenWrt support.

---

## Hardware Specifications

| Feature | Specification |
|---|---|
| CPU | Qualcomm IPQ4019 @ 716 MHz (quad-core ARMv7) |
| RAM | 512 MB |
| Flash | 256 MB NAND |
| Switch | QCA8075 (5-port) |
| Radios | IPQ4019 (2.4GHz + 5GHz) + QCA9884 (5GHz PCIe) |
| USB | USB 3.0 |
| Codename | Dallas / Lion |

---

## Why This Router Is Interesting for OpenWrt

With 512MB RAM and 256MB flash, the MR9000 is exceptional value for OpenWrt. Most home routers struggle with 64-128MB RAM. This device can comfortably run AdGuard Home, WireGuard VPN, Samba NAS, and much more simultaneously without breaking a sweat.

---

## Understanding the Dual Boot System — IMPORTANT

Before doing anything, understanding this is essential. The MR9000 has **two completely separate firmware partitions** — think of it as two independent storage slots, each capable of holding a full router firmware.

```
Partition 1: linux (mtd10) + rootfs (mtd11)   ← primary slot
Partition 2: linux2 (mtd12) + rootfs2 (mtd13) ← alternate slot
```

**How it works:**
- The router always boots from one partition at a time
- When you upload firmware through the web UI, it writes to the **other** partition — not the one currently running. This is a deliberate safety mechanism.
- If the new firmware fails to boot, the router can fall back to the working partition
- This means you effectively cannot permanently brick this router through normal firmware flashing

**Switching between partitions:**
Performing **3-4 consecutive power cycles** (power off and on 3-4 times within about 3 seconds each cycle) triggers the auto-recovery mechanism and switches to the other partition. This is the "rescue" procedure if something goes wrong.

**Switching from LuCI (once OpenWrt is running):**
Install the `luci-app-advanced-reboot` package. This adds an Advanced Reboot page under System in LuCI that lets you select which partition to boot from with a single click — much more convenient than the power cycle method.

```bash
# Install from OpenWrt shell
opkg update
opkg install luci-app-advanced-reboot
```

**The recommended safe strategy used in this guide:**
- Keep Linksys OEM firmware on Partition 1 — your permanent safety net
- Install and experiment with OpenWrt on Partition 2
- Only when you are confident OpenWrt is fully working and stable, consider overwriting Partition 1

---

## LED Behaviour Guide

Understanding what the LEDs mean helps diagnose what state the router is in:

| Situation | Top LED | LAN LEDs |
|---|---|---|
| Linksys OEM booting | Blue → off → on → **Violet** | Sequential light-up then Orange |
| Linksys OEM running | **Violet** | Orange |
| OpenWrt booting | Blue flashing | Sequential light-up (green only) |
| OpenWrt running | **Green** | Green (no orange) |
| DD-WRT running | Blue | Green (no orange) |
| Firmware not loading / stuck / panic | Blue solid (stays blue, never changes) | Green blinks on ping only |

**Important notes:**
- The top LED colour is the most reliable indicator: **Violet = OEM**, **Green = OpenWrt**, **Blue = DD-WRT or stuck**
- Orange LAN LEDs are controlled by proprietary Linksys code — they do not work in OpenWrt or DD-WRT. This is cosmetic only and does not affect networking
- The sequential LAN LED lighting during boot DOES happen in OpenWrt — but only green, not orange
- If the router is stuck (blue LED solid, no progress), the green LAN LED will still blink when you ping `192.168.1.1` — this is the switch chip responding at hardware level independently of the CPU. **Do not interpret this as a successful boot**

---

## The Problem — Why It's Not Officially Supported

### 1. No Official OpenWrt Firmware Exists
The MR9000 is listed as "not yet supported" on the OpenWrt wiki. It is similar to the EA8300 but not identical, and using EA8300 firmware does not work — see the technical diagnosis section below.

### 2. The Kernel Size Wall
OpenWrt versions newer than 22.x require a larger kernel partition than the factory default. The stock U-Boot environment has `kernsize=300000` (3MB), but kernels from 23.x onwards are larger than 3MB and will not boot with this setting. It must be changed to `kernsize=500000` (5MB) before flashing anything newer than 22.x. This requires a working shell on the router first — creating a catch-22 situation.

### 3. EA8300 Firmware Fails on MR9000
Even with the correct kernel size, EA8300 OpenWrt images fail to boot on the MR9000 for hardware reasons detailed in the technical diagnosis section below.

---

## Technical Diagnosis — Why EA8300 OpenWrt Fails on MR9000

### The Switch Chip Difference
The EA8300 uses a **QCA8072** switch chip (2-port PHY), while the MR9000 uses a **QCA8075** switch chip (5-port PHY). These are completely different chips with different register layouts, different MDIO addresses, and different initialisation sequences.

### What Actually Happens When You Flash EA8300 OpenWrt on MR9000

1. U-Boot loads correctly — blue LED turns on
2. The Linux kernel starts loading
3. The kernel attempts to initialise the switch chip using the EA8300 Device Tree (DTS), which describes a QCA8072
4. The kernel finds a QCA8075 where it expects a QCA8072
5. The switch driver fails — causing a kernel panic or silent halt
6. The router appears stuck: blue LED on, green LAN LED blinks on ping, but no IP connectivity, no SSH, nothing

**This is why every version (22.x, 23.x, 24.x, even 25.x with EA8300 image) shows identical stuck behaviour** — the switch chip failure happens before any networking can start.

### The Architecture Difference

| Version | Kernel | Switch Architecture | MR9000 QCA8075 Support |
|---|---|---|---|
| 22.x | 5.10 | swconfig (old) | Partial — driver exists but wrong architecture |
| 23.x | 5.15 | DSA (new) | Incomplete |
| 24.x | 5.15 | DSA | Incomplete |
| 25.x | 6.12 | DSA | ✅ Full support |

Full QCA8075 support with proper DSA architecture only landed in OpenWrt 25.x with kernel 6.12. MR9000-specific builds include the correct Device Tree and QCA8075 DSA driver support — EA8300 builds do not.

---

## Flash Memory Layout — Full Address Map

From running `cat /proc/mtd` on a live DD-WRT shell and mapping addresses:

```
Address Range              Name         Size    Purpose
0x00000000 - 0x00100000   sbl1          1MB    Secondary Boot Loader (low-level HW init)
0x00100000 - 0x00200000   mibib         1MB    Qualcomm partition table
0x00200000 - 0x00300000   qsee          1MB    Qualcomm Secure Execution Environment (TrustZone)
0x00300000 - 0x00380000   cdt         512KB    Hardware Configuration Data Table
0x00380000 - 0x00400000   appsblenv   512KB    U-Boot internal configuration
0x00400000 - 0x00480000   ART         512KB    Atheros Radio Test (WiFi calibration — DO NOT ERASE)
0x00480000 - 0x00680000   appsbl        2MB    U-Boot (the main bootloader)
0x00680000 - 0x00700000   u_env       512KB    User environment variables (boot_part, kernsize etc)
0x00700000 - 0x00740000   s_env       256KB    Boot attempt counter (drives auto-recovery/partition switching)
0x00740000 - 0x00780000   devinfo     256KB    Device serial number and hardware identifiers
0x00780000 - 0x05f80000   linux        88MB    Partition 1: kernel (first 5MB) + rootfs
0x00c80000 - 0x05f80000   rootfs       83MB    Partition 1: rootfs (sub-partition of linux)
0x05f80000 - 0x0b680000   linux2       87MB    Partition 2: kernel (first 5MB) + rootfs2
0x06480000 - 0x0b680000   rootfs2      82MB    Partition 2: rootfs (sub-partition of linux2)
0x0b680000 - 0x0b780000   nvram         1MB    User settings (SSID, passwords etc)
0x0b780000 - 0x0b880000   oops          1MB    Kernel panic/crash logs
0x0b880000 - 0x0ff00000   [unused]    ~70MB    Unallocated flash space (see discovery below)
0x0ff00000 - 0x10000000   [reserve]     1MB    Bad block reserve area
```

**Total flash: 256MB (0x10000000)**

### The Boot Chain
```
CPU internal ROM (PBL — hardwired in silicon, invisible)
  → sbl1 (initialises RAM and NAND controller)
    → appsbl / U-Boot (full bootloader with commands and network stack)
      → Linux kernel (from linux or linux2 partition)
        → rootfs (OpenWrt / OEM / DD-WRT)
```

### Understanding kernsize
`kernsize=500000` means U-Boot reads 0x500000 bytes (5MB) from the kernel partition start before looking for the rootfs:

```
linux partition start:  0x00780000
rootfs start:           0x00c80000
Difference:             0x00500000 = 5MB = kernsize 500000 ✓
```

---

## Discovery: ~70MB of Unallocated Flash Space

The MR9000 (and the entire Linksys EA/MR xx8300 family including EA8300, MR8300, EA8250) has approximately **70MB of completely unallocated flash space** starting at `0x0b880000`.

Linksys apparently copied the EA8300 partition layout to all devices in this family without using the extra space on larger-flash variants. This space is available for custom partitions.

**How this was discovered:** DD-WRT was found to have created a partition called `ddwrt` (mtd16) at exactly `0x0b880000 - 0x0ff00000`. DD-WRT quietly occupied this unused space during installation without documenting it anywhere. Mapping all partition addresses revealed the unallocated space and confirmed DD-WRT's partition does not overlap with anything existing — it simply claimed previously unused flash.

**Potential uses for this space:**
- A third boot partition (triple boot — OEM + OpenWrt + DD-WRT or experimental builds)
- Persistent storage that survives firmware flashing
- Extended package overlay
- Dedicated logging partition

---

## Community Credits

This router would not be installable without the work of these community contributors:

- **@rekados** — Submitted the MR9000 BDF (Board Data Files / WiFi radio calibration data) to the official OpenWrt firmware repository, enabling correct WiFi operation on all three radios. [PR #114](https://github.com/openwrt/firmware_qca-wireless/pull/114)
- **@RolandoMagico** — Maintains MR9000-specific OpenWrt builds and releases on GitHub. [Releases](https://github.com/RolandoMagico/openwrt-build/releases)
- **@sshel** — Maintains a comprehensive build server with MR9000-specific firmware from version 22.03.5 through 25.12.x, including factory.bin files essential for initial installation. [sshel.inf.ua/mr9000](https://sshel.inf.ua/mr9000/). The builds hosted here were used for the successful installation documented in this guide.
- **@robimarko** — OpenWrt maintainer who reviewed and merged the MR9000 support work into the official repository
- **@NoTengoBattery**, **@GillyMoMo**, **@drandyhaas**, **@reka** and many others who contributed to the [OpenWrt forum thread](https://forum.openwrt.org/t/support-for-the-linksys-mr9000/54377/) since 2020 and kept this project alive

---

## Firmware Downloads

### Recommended: sshel.inf.ua builds (includes factory.bin)

| Version | Directory | Notes |
|---|---|---|
| 22.03.5 | [sshel.inf.ua/mr9000/22.03.5](https://sshel.inf.ua/mr9000/?dir=mr9000%2F22.03.5) | Bootstrap step — use factory.bin |
| 25.12.2 | [sshel.inf.ua/mr9000/25.12.2](https://sshel.inf.ua/mr9000/?dir=mr9000%2F25.12.2) | Latest stable — use factory.bin |
| All versions | [sshel.inf.ua/mr9000](https://sshel.inf.ua/mr9000/) | Full list |

### Alternative: RolandoMagico builds (sysupgrade only — for upgrading existing OpenWrt)
[github.com/RolandoMagico/openwrt-build/releases](https://github.com/RolandoMagico/openwrt-build/releases)

---

## Installation Methods

There are two serial-free installation paths. Both achieve the same result.

---

### Method A: OpenWrt 22.x Bootstrap (Recommended — Simpler)

Flash a working MR9000-specific 22.x image first to get a shell, increase the kernel size, then upgrade to 25.x. The OEM firmware stays safely on the other partition throughout.

**What you need:**
- A PC with Ethernet port (or USB to Ethernet adapter)
- An Ethernet cable
- `openwrt-22.03.5-ipq40xx-generic-linksys_mr9000-squashfs-factory.bin` — download from [sshel.inf.ua](https://sshel.inf.ua/mr9000/?dir=mr9000%2F22.03.5)
- `openwrt-25.12.2-ipq40xx-generic-linksys_mr9000-squashfs-factory.bin` — download from [sshel.inf.ua](https://sshel.inf.ua/mr9000/?dir=mr9000%2F25.12.2)

**Steps:**

**Step 1 — Boot into Linksys OEM firmware**

Connect your PC to one of the LAN ports on the back of the router with an Ethernet cable. If not already on OEM firmware, perform 3-4 power cycles to switch back to the OEM partition — the top LED will be violet when OEM is running.

**Step 2 — Access the hidden firmware update page**

Open your browser and go to:
```
https://192.168.1.1/fwupdate.html
```
> This is the MR series hidden firmware update page. The regular firmware update page in the Linksys web interface may refuse third-party firmware. The hidden page accepts it.

You will likely see a certificate warning — click through it (click Advanced → Proceed). Login with your router admin password (default: `admin`).

**Step 3 — Flash OpenWrt 22.03.5 MR9000**

Select `openwrt-22.03.5-ipq40xx-generic-linksys_mr9000-squashfs-factory.bin` and click Update. **Do not touch the router for at least 4 minutes** — do not close the browser, do not turn the router off.

After flashing, the router will reboot. The top LED will turn green — this means OpenWrt is running.

**Step 4 — Connect to OpenWrt 22.x via SSH**

Set your PC's network adapter to a static IP address:
- IP: `192.168.1.2`
- Subnet mask: `255.255.255.0`
- Gateway: `192.168.1.1`

Then connect via SSH. **Windows users** can use [PuTTY](https://www.putty.org/) (free download):
- Open PuTTY
- Host Name: `192.168.1.2` → change to `192.168.1.1`
- Port: `22`
- Connection type: SSH
- Click Open
- When asked for username type `root` and press Enter
- When asked for password just press Enter (no password on fresh OpenWrt install)

**Linux/Mac users:**
```bash
ssh root@192.168.1.1
# Press Enter when asked for password
```

**Step 5 — Increase the kernel partition size**

Once you have a shell prompt, type these commands:

```bash
fw_setenv kernsize 500000
```
Then verify it was set correctly:
```bash
fw_printenv kernsize
```
It should return: `kernsize=500000`

Then reboot:
```bash
reboot
```

**Step 6 — Flash OpenWrt 25.x**

After the router reboots (top LED green again), open your browser and go to:
```
http://192.168.1.1
```

Login to LuCI (the OpenWrt web interface). Go to **System → Backup / Flash Firmware → Flash new firmware image**.

Select `openwrt-25.12.2-ipq40xx-generic-linksys_mr9000-squashfs-factory.bin` and flash it.

> **Important:** When asked whether to keep settings, choose **NO / Do not preserve configuration**. The network architecture changed between 22.x and 25.x and keeping old settings may cause problems.

Wait 4 minutes. The router will reboot with OpenWrt 25.12.2 running.

**Step 7 — Done!**

The top LED will be green. Open `http://192.168.1.1` in your browser — you should see the OpenWrt LuCI interface showing:
- Model: Linksys MR9000 (Dallas)
- Firmware: OpenWrt 25.12.2
- Kernel: 6.12.x
- All three WiFi radios detected

> **At this point:** Linksys OEM is still safely on Partition 1. OpenWrt 25.x is on Partition 2. You can switch back to OEM at any time using 3-4 power cycles.

---

### Method B: DD-WRT Bootstrap (Alternative)

This method was discovered before the MR9000-specific 22.x builds were found. It remains valid and also reveals that DD-WRT supports the MR9000 if you ever want to run it.

**What you need:**
- `factory-to-ddwrt.bin` for MR9000 from [DD-WRT router database](https://dd-wrt.com/support/router-database/?model=linksys+mr9000)
- `openwrt-25.12.2-ipq40xx-generic-linksys_mr9000-squashfs-factory.bin` from [sshel.inf.ua](https://sshel.inf.ua/mr9000/?dir=mr9000%2F25.12.2)

**Steps:**

**Step 1 — Flash DD-WRT**

Go to `https://192.168.1.1/fwupdate.html`, login with `admin/admin`, select the `factory-to-ddwrt.bin` file for MR9000 and click Update. Wait 4 minutes.

**Step 2 — Set up DD-WRT**

After reboot, go to `http://192.168.1.1`. DD-WRT will ask you to set a username and password — do this before proceeding.

**Step 3 — Increase kernel partition size**

In the DD-WRT web interface go to **Administration → Commands**. In the text box type:
```
fw_setenv kernsize 500000
```
Click **Run Commands**.

Then verify by running:
```
fw_printenv kernsize
```
It should return `kernsize=500000`.

**Step 4 — Flash OpenWrt 25.x**

In DD-WRT go to **Administration → Firmware Upgrade**. Select `openwrt-25.12.2-ipq40xx-generic-linksys_mr9000-squashfs-factory.bin` and flash it. Wait 4 minutes.

**Step 5 — Done!**

OpenWrt 25.12.2 will boot with the green top LED.

---

## Verifying the Installation

After booting OpenWrt 25.x, connect via SSH and verify everything is correct:

**Windows:** Open PuTTY, connect to `192.168.1.1` on port 22, login as `root` with no password.

**Linux/Mac:**
```bash
ssh root@192.168.1.1
```

Run these checks:

```bash
# Confirm device is correctly identified
cat /tmp/sysinfo/model
# Expected: Linksys MR9000 (Dallas)

# Confirm kernel version
uname -r
# Expected: 6.12.x

# Confirm kernsize is still correct
fw_printenv kernsize
# Expected: kernsize=500000

# Confirm partition layout
cat /proc/mtd

# Check available RAM (should show ~490MB)
free -m

# Check which partition you are on
fw_printenv boot_part
# 1 = OEM partition, 2 = your OpenWrt partition
```

---

## Installing luci-app-advanced-reboot (Recommended)

This package adds a proper partition management page to LuCI, making it easy to switch between partitions without power cycling.

```bash
opkg update
opkg install luci-app-advanced-reboot
```

After installing, go to **System → Advanced Reboot** in LuCI. You can see both partitions and choose which one to boot from — much more convenient than the 3-4 power cycle method.

---

## Upgrading to Newer Versions

Once OpenWrt is running, future upgrades are straightforward via LuCI or command line:

```bash
# Download new sysupgrade image
cd /tmp
wget https://sshel.inf.ua/mr9000/25.12.x/openwrt-25.12.x-ipq40xx-generic-linksys_mr9000-squashfs-sysupgrade.bin

# Flash it (omit -n if you want to keep settings)
sysupgrade -n /tmp/openwrt-*.bin
```

Or use LuCI → System → Backup/Flash Firmware.

---

## Quick Summary for Non-Technical Users

This section gives you everything you need to install OpenWrt on your MR9000 step by step, even if you are not familiar with networking or Linux. **Read the disclaimer at the top before proceeding.**

**What is OpenWrt?** It is a replacement firmware (operating system) for your router that gives you much more control and features than the factory Linksys firmware.

**Is it safe?** The MR9000 has two firmware slots. The process below only touches one slot — your original Linksys firmware stays untouched on the other slot. If anything goes wrong, you can get back to Linksys by turning the router off and on 3-4 times quickly.

**What you need:**
- A Windows PC (or Mac/Linux)
- An Ethernet cable
- Download these two files before starting:
  - **File 1 (22.x):** Go to [this link](https://sshel.inf.ua/mr9000/?dir=mr9000%2F22.03.5) and download `openwrt-22.03.5-ipq40xx-generic-linksys_mr9000-squashfs-factory.bin`
  - **File 2 (25.x):** Go to [this link](https://sshel.inf.ua/mr9000/?dir=mr9000%2F25.12.2) and download `openwrt-25.12.2-ipq40xx-generic-linksys_mr9000-squashfs-factory.bin`
  - **PuTTY (SSH client for Windows):** Download free from [putty.org](https://www.putty.org/)

**Step 1** — Connect your PC to the router with an Ethernet cable (any LAN port on the back). Make sure the router is showing the violet top LED (Linksys OEM running).

**Step 2** — Open your browser and type this address exactly: `https://192.168.1.1/fwupdate.html`
- You will see a security warning — click Advanced and then Proceed (or similar, depending on your browser)
- Login with password `admin` (or your custom password if you changed it)

**Step 3** — Click Choose File (or Browse), select **File 1** (the 22.x file), click Update. Wait 5 minutes without touching anything. The top LED will turn green when done.

**Step 4** — Open PuTTY on your Windows PC:
- In the "Host Name" box type: `192.168.1.1`
- Make sure Port is `22` and SSH is selected
- Click Open
- If a security warning appears click Accept/Yes
- Type `root` when asked for username and press Enter
- Just press Enter when asked for password (there is no password)
- You should see a command prompt

**Step 5** — Type this command exactly and press Enter:
```
fw_setenv kernsize 500000
```
Then type this to verify:
```
fw_printenv kernsize
```
It should say `kernsize=500000`. Then type:
```
reboot
```
And press Enter. Close PuTTY.

**Step 6** — Wait 2 minutes for the router to reboot (top LED goes green again). Open your browser and go to `http://192.168.1.1`. You will see the OpenWrt interface. Go to **System → Backup / Flash Firmware**, click **Flash new firmware image**, select **File 2** (the 25.x file). When asked about keeping settings choose **No**. Click **Proceed**. Wait 5 minutes.

**Step 7** — Done! Go to `http://192.168.1.1`. You should see OpenWrt 25.12.2 running on your Linksys MR9000. Your original Linksys firmware is still safely stored and you can go back to it at any time by turning the router off and on 3-4 times quickly.

---

## Related Resources

- [OpenWrt MR9000 Wiki Page](https://openwrt.org/inbox/toh/linksys/mr9000)
- [OpenWrt Forum Thread (2020–present)](https://forum.openwrt.org/t/support-for-the-linksys-mr9000/54377/)
- [OpenWrt EA8300 Wiki Page](https://openwrt.org/toh/linksys/ea8300)
- [GitHub PR: Add support for Linksys MR9000](https://github.com/openwrt/openwrt/pull/22744)
- [GitHub PR: Add Linksys MR9000 BDF](https://github.com/openwrt/firmware_qca-wireless/pull/114)
- [My DSL-500T Recovery Guide](https://github.com/xeXistenZx/Recovering-bricked-Dlink-500T-with-Adam2-bootloader)

---

*Guide by [@xeXistenZx](https://github.com/xeXistenZx) — documented from personal experience, May 2026*

*Special thanks to the entire MR9000 OpenWrt community, especially @rekados, @RolandoMagico, @sshel, @robimarko, @NoTengoBattery, @GillyMoMo, @drandyhaas, @reka and everyone in the forum thread who kept this project alive since 2020.*
