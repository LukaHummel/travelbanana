# travelbanana
Travelrouter with integrated Android TV based on Banana PI BPI-M5 Pro
# BPI‑M5 Pro Travel Router + Android TV Stack

This document describes how to build a **travel router with integrated Android TV** on a **Banana Pi BPI‑M5 Pro (RK3576)** using:

- **Debian/Armbian host**
- **Weston** (Wayland compositor) on DRM/KMS
- **Waydroid** with **Waydroid‑ATV** images
- **OpenWrt in an LXC container** as the router (WAN/LAN, DHCP, firewall, DNS)

> **Scope:** The host only does hypervisor, display, and Android; **all routing** (NAT, DHCP, firewall) is inside OpenWrt‑LXC.

---

## 1. Hardware and OS References

Use these as your authoritative references for the **BPI‑M5 Pro**:

- BPI‑M5 Pro docs (RK3576, RAM, eMMC, OS support):  
  https://docs.banana-pi.org/en/BPI-M5/BananaPi_BPI-M5_Pro [web:1]  
- Forum announcement (more SoC/board details, OS mentions):  
  https://forum.banana-pi.org/t/banana-pi-bpi-m5-pro-with-rockchip-rk3576-max-support-16g-ram-and-128g-emmc/17976 [web:36]  
- Shop/feature overview (RAM/eMMC, dual GbE, Wi‑Fi 6, HDMI, etc.):  
  https://www.youyeetoo.com/products/banana-pi-bpi-m5-pro [web:33]  
- Armbian forum thread for M5 Pro (kernel/images status):  
  https://forum.armbian.com/topic/38891-banana-pi-bpi-m5-pro-with-rockchip-rk3576/ [web:34]  
- Independent board summary (octa‑core RK3576, Mali‑G52, dual GbE, Wi‑Fi 6):  
  https://linuxgizmos.com/90351-2/ [web:76]

---

## 2. Target Architecture

```text
BPI-M5 Pro (RK3576) ─ Debian/Armbian host ─ systemd

 ├─ Weston (Wayland compositor, DRM/KMS, kiosk)
 │    └─ Waydroid container
 │         └─ Android TV images (Waydroid-ATV)
 │             (HDMI output)
 │
 └─ LXC (host)
      └─ OpenWrt container (privileged)
           ├─ WAN: veth0 ↔ br-wan ↔ physical WAN NIC
           └─ LAN: veth1 ↔ br-lan ↔ physical LAN NIC (+ Wi-Fi AP)
```

- Host networking: **bridges only** (`br-wan`, `br-lan`).
- OpenWrt container: **full router** (NAT, DHCP, DNS, firewall, Wi‑Fi AP).

---

## 3. Base OS on BPI‑M5 Pro

### 3.1 Choose an OS

Use an **Armbian Debian‑based image** for BPI‑M5 Pro or the closest supported variant as described in the Armbian forum thread.[web:34]

- Armbian boards page (M5 family is here; follow links from the Pro thread):  
  https://www.armbian.com/boards/bananapim5 [web:75]  
- M5 Pro thread (for specific images / DTB notes):  
  https://forum.armbian.com/topic/38891-banana-pi-bpi-m5-pro-with-rockchip-rk3576/ [web:34]

### 3.2 Install to eMMC

Typical flow (adapt to exact image and device names):

1. Flash Armbian image to SD.
2. Boot from SD, log in.
3. Use `armbian-install`/`armbian-config` or manual copy to install to eMMC (as described in the M5 Pro thread). [web:34]

### 3.3 Verify hardware

Before layering anything else:

- Check dual GbE (`ip link`).
- Check Wi‑Fi 6/BT (e.g. `iw list`, `rfkill list`).
- Confirm HDMI output + audio.
- Confirm panfrost for Mali‑G52 (kernel logs + `glxinfo`/`kmscube`):  
  - Armbian Rockchip tree/panfrost enablement:  
    https://github.com/armbian/linux-rockchip [web:50]  
    https://github.com/armbian/linux-rockchip/pull/249 [web:48]

---

## 4. LXC on Debian/Armbian Host

### 4.1 LXC docs

- LXC documentation hub:  
  https://linuxcontainers.org/lxc/documentation/ [web:81]  
- LXC homepage:  
  https://linuxcontainers.org [web:87]  
- LXC man page (config reference):  
  https://man7.org/linux/man-pages/man7/lxc.7.html [web:85]  
- Debian‑focused guide (pattern for unprivileged containers and bridges):  
  https://blog.michaelkelly.org/2023/09/lxc-containers-on-debian-part-1-setup/ [web:83]

### 4.2 Install and basic check

```bash
sudo apt update
sudo apt install lxc bridge-utils
sudo lxc-checkconfig
```

You want:

- Namespaces, cgroup v2, veth, bridge, etc. all **enabled**.[web:85]

For OpenWrt specifically, use a **privileged container**, as that matches most working how‑tos and reduces surprises.

---

## 5. OpenWrt in LXC

### 5.1 OpenWrt LXC documentation

Primary guide:

- **OpenWrt in LXC containers**:  
  https://openwrt.org/docs/guide-user/virtualization/lxc [web:56]

It documents two approaches:

- Using `lxc-create -t download` with an OpenWrt image where available.[web:56]  
- Using a manually downloaded rootfs tarball and custom config.[web:56]

For arm64, you may need the manual method; check the wiki first.

### 5.2 Creating the OpenWrt container

#### Option A: via `download` template (if OpenWrt appears)

```bash
sudo lxc-create -n openwrt -t download -- -d openwrt -a arm64
```

Choose a suitable release if prompted.[web:56]

#### Option B: via rootfs extraction (manual)

Follow the **“Via rootfs extraction”** section in the OpenWrt LXC doc:[web:56]

1. Create container dir:

```bash
sudo mkdir -p /var/lib/lxc/openwrt/rootfs
cd /var/lib/lxc/openwrt
```

2. Download OpenWrt aarch64 rootfs tarball (see OpenWrt downloads, per doc).[web:56]

3. Extract into `rootfs/`:

```bash
sudo tar xvf openwrt-*-rootfs.tar.gz -C rootfs
```

4. Create `/var/lib/lxc/openwrt/config` (see next subsection).

### 5.3 Example `config` for OpenWrt‑LXC (WAN + LAN)

Minimal example (adapt to your paths):

```ini
# /var/lib/lxc/openwrt/config

# Container basics
lxc.uts.name = openwrt
lxc.rootfs.path = dir:/var/lib/lxc/openwrt/rootfs

# Privileged container
lxc.idmap =

# TTY / console
lxc.tty.max = 4
lxc.pty.max = 1024
lxc.console.path = none

# Capabilities (OpenWrt expects broad capability set)
lxc.cap.drop =

# Init
lxc.init.cmd = /sbin/init

# Network: WAN (eth0 in OpenWrt)
lxc.net.0.type = veth
lxc.net.0.link = br-wan
lxc.net.0.flags = up
lxc.net.0.hwaddr = 00:16:3e:aa:bb:01

# Network: LAN (eth1 in OpenWrt)
lxc.net.1.type = veth
lxc.net.1.link = br-lan
lxc.net.1.flags = up
lxc.net.1.hwaddr = 00:16:3e:aa:bb:02
```

- `br-wan` and `br-lan` are bridges that will exist on the host (next section).
- Inside OpenWrt you’ll configure `eth0` as `wan` and `eth1` as `lan` as usual.

For more inspiration and gotchas (even if Proxmox‑specific):

- Gist: OpenWrt 23.05 LXC container (Proxmox):  
  https://gist.github.com/suuhm/053f819b000bee4af922d66ff6c5d32e [web:73]  
- “Installing OpenWRT on top of LXC in Proxmox” (discussion on support/limitations):  
  https://lowendspirit.com/discussion/2266/installing-openwrt-on-top-of-lxc-in-proxmox [web:69]

---

## 6. Host Networking (Bridges Only)

Goal: host is just a **bridge and hypervisor**; OpenWrt does all routing.

### 6.1 Bridging physical NICs

Assume:

- `eth0` = physical WAN port
- `eth1` = physical LAN port

Create two Linux bridges (you can use systemd‑networkd, NetworkManager, or `ip` commands).

Example with systemd‑networkd style (conceptual):

```ini
# /etc/systemd/network/10-br-wan.netdev
[NetDev]
Name=br-wan
Kind=bridge

# /etc/systemd/network/20-br-wan-eth0.network
[Match]
Name=eth0

[Network]
Bridge=br-wan

# /etc/systemd/network/10-br-lan.netdev
[NetDev]
Name=br-lan
Kind=bridge

# /etc/systemd/network/20-br-lan-eth1.network
[Match]
Name=eth1

[Network]
Bridge=br-lan

# Optional: Host management IP on br-lan
# /etc/systemd/network/30-br-lan.network
[Match]
Name=br-lan

[Network]
Address=192.168.1.2/24
Gateway=192.168.1.1
```

- **No IP on `br-wan`** → full pass‑through to the OpenWrt WAN interface.  
- Optional static IP on `br-lan` → host is reachable for SSH even if OpenWrt is down.

Attach LXC veth interfaces via `lxc.net.*.link = br-*` (section 5.3).

Additional discussions of OpenWrt‑LXC networking patterns:

- Reddit: “How to configure OpenWrt networking with LXC?”  
  https://www.reddit.com/r/openwrt/comments/1engfcz/how_to_configure_openwrt_networking_with_lxc/ [web:74]

---

## 7. Weston (DRM/KMS) on Host

### 7.1 Docs

- Weston docs:  
  https://wayland.pages.freedesktop.org/weston/  
- NVIDIA Jetson Weston guide (good DRM/KMS and kiosk explanation):  
  https://docs.nvidia.com/jetson/archives/r38.4/DeveloperGuide/SD/WindowingSystems/WestonWayland.html [web:66]  
- Weston repo/mirror:  
  https://github.com/intel/Intel-Distribution-of-Weston [web:60]

### 7.2 Install Weston

```bash
sudo apt install weston
```

Create a dedicated user, e.g.:

```bash
sudo adduser tvuser
```

Configure **autologin** on tty1 (via `getty@tty1.service` override), then use a user‑level systemd unit to start Weston.

### 7.3 Example user unit for Weston (systemd --user)

```ini
# ~/.config/systemd/user/weston.service

[Unit]
Description=Weston Wayland Compositor (DRM)
After=graphical-session.target
Wants=graphical-session.target

[Service]
Type=simple
Environment= XDG_RUNTIME_DIR=/run/user/%U
ExecStart=/usr/bin/weston --backend=drm-backend.so --tty=1 --idle-time=0
Restart=on-failure

[Install]
WantedBy=default.target
```

Enable as `tvuser`:

```bash
systemctl --user enable weston.service
systemctl --user start weston.service
```

---

## 8. Waydroid + Waydroid‑ATV

### 8.1 Base Waydroid install

Docs:

- Waydroid install on desktops:  
  https://docs.waydro.id/usage/install-on-desktops [web:67]  
- GitHub doc:  
  https://github.com/waydroid/docs/blob/master/usage/install-on-desktops.md [web:61]

On Debian/Armbian:

```bash
curl -s https://repo.waydro.id | sudo bash      # add repo (add -s bookworm/trixie if needed)
sudo apt install waydroid
sudo waydroid init
```

This will pull default system/vendor images and configure the container.[web:67][web:61]

### 8.2 Swap in Android TV (Waydroid‑ATV)

Reference release (example – use latest):

- Waydroid‑ATV builds with installation notes:  
  https://newreleases.io/project/github/WayDroid-ATV/waydroid-androidtv-builds/release/20250327 [web:26]

From the release notes:

1. Download `lineage-20.0-*-UNOFFICIAL-WaydroidATV_*.zip`. [web:26]  
2. Extract `system.img` and `vendor.img` from the ZIP. [web:26]  
3. Place in Waydroid extra images directory:

   ```bash
   sudo mkdir -p /etc/waydroid-extra/images/
   sudo cp system.img /etc/waydroid-extra/images/system.img
   sudo cp vendor.img /etc/waydroid-extra/images/vendor.img
   ```

4. Re‑init Waydroid:

   ```bash
   sudo waydroid init -f
   ```

Waydroid will now use the ATV images.[web:26]

Background context thread:

- “Finally got Android TV compiled for Waydroid” (includes repo link):  
  https://www.reddit.com/r/waydroid/comments/1emc5kv/finally_got_android_tv_compiled_for_waydroid/ [web:68]

### 8.3 Autostart Waydroid full UI under Weston

Once Weston is running as `tvuser`:

```bash
sudo waydroid container start      # usually root needed for container start
waydroid show-full-ui              # run as tvuser, within Weston
```

You can wrap this in a script and a user systemd unit:

**Script:**

```bash
# /usr/local/bin/waydroid-atv-start.sh
#!/bin/bash
set -e

# Ensure container is up (requires sudo; handle via sudoers or helper)
sudo waydroid container start

# Give container a moment
sleep 5

# Launch full UI
waydroid show-full-ui
```

```bash
sudo chmod +x /usr/local/bin/waydroid-atv-start.sh
```

**User unit (tvuser):**

```ini
# ~/.config/systemd/user/waydroid-atv.service

[Unit]
Description=Waydroid Android TV Full UI
After=weston.service
Requires=weston.service

[Service]
Type=simple
Environment= WAYLAND_DISPLAY=wayland-0
ExecStart=/usr/local/bin/waydroid-atv-start.sh
Restart=on-failure

[Install]
WantedBy=default.target
```

Enable:

```bash
systemctl --user enable waydroid-atv.service
systemctl --user start waydroid-atv.service
```

Now boot → `tvuser` autologin → Weston → Waydroid‑ATV full screen.

---

## 9. OpenWrt Configuration Inside the Container

Once the container is up (e.g. `sudo lxc-start -n openwrt -F`):

1. Access the OpenWrt shell via LXC console or SSH (if configured).
2. In `/etc/config/network`:
   - `eth0` → `wan` (DHCP client, or static if you’re upstreaming to another router).  
   - `eth1` → `lan` (static, e.g. 192.168.1.1/24, running DHCP server).  

3. Configure firewall, DNS, etc. as you would on a bare‑metal OpenWrt router.

Use the OpenWrt LXC doc as reference for specifics (mounts, fstab, etc.):  
https://openwrt.org/docs/guide-user/virtualization/lxc [web:56]

---

## 10. Useful Related References

- LXC generic docs (conceptual + API):  
  https://linuxcontainers.org/lxc/documentation/ [web:81]  
- Container image server (if you later want LXD‑style images):  
  https://images.linuxcontainers.org [web:88]  
- ArchWiki on Linux Containers for conceptual patterns:  
  https://wiki.archlinux.org/title/Linux_Containers [web:86]  

- OpenWrt LXC networking discussion:  
  https://www.reddit.com/r/openwrt/comments/1engfcz/how_to_configure_openwrt_networking_with_lxc/ [web:74]  

---

## 11. Boot Flow Summary

1. **Bootloader → Armbian/Debian on RK3576**.[web:1][web:34]  
2. systemd:
   - Brings up `br-wan`, `br-lan`, and NICs.  
   - Starts LXC and **OpenWrt** container (router).  
3. On tty1, `tvuser` auto‑login:
   - `systemd --user` starts **Weston** (DRM).  
   - Weston runs `waydroid-atv.service` which starts the Waydroid container and launches **Android TV UI** full‑screen.  

Your box now behaves as a travel router with a full Android TV front‑end on HDMI, all in one device.
