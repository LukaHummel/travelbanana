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

Authoritative references for the **BPI‑M5 Pro**:

- Banana Pi BPI‑M5 Pro docs (RK3576, RAM, eMMC, OS support):  
  https://docs.banana-pi.org/en/BPI-M5/BananaPi_BPI-M5_Pro

- Forum announcement (more SoC/board details, OS mentions):  
  https://forum.banana-pi.org/t/banana-pi-bpi-m5-pro-with-rockchip-rk3576-max-support-16g-ram-and-128g-emmc/17976

- Shop/feature overview (RAM/eMMC, dual GbE, Wi‑Fi 6, HDMI, etc.):  
  https://www.youyeetoo.com/products/banana-pi-bpi-m5-pro

- Armbian forum thread for M5 Pro (kernel/images status):  
  https://forum.armbian.com/topic/38891-banana-pi-bpi-m5-pro-with-rockchip-rk3576/

- Board overview article (octa‑core RK3576, Mali‑G52, dual GbE, Wi‑Fi 6, etc.):  
  https://linuxgizmos.com/90351-2/

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

Use an **Armbian Debian‑based image** for the BPI‑M5 Pro or the closest supported variant, as described in the Armbian forum thread:

- Armbian boards overview (M5 family):  
  https://www.armbian.com/boards/bananapim5

- M5 Pro thread (for specific images / DTB notes):  
  https://forum.armbian.com/topic/38891-banana-pi-bpi-m5-pro-with-rockchip-rk3576/

### 3.2 Install to eMMC

Typical flow (adapt to exact image and device names):

1. Flash the Armbian image for BPI‑M5 Pro (or closest supported) to an SD card.
2. Boot from SD, log in.
3. Use `armbian-install` / `armbian-config` or manual `dd`/`rsync` to install to eMMC, following guidance in the M5 Pro thread.

### 3.3 Verify hardware

Before layering anything else:

- Check dual GbE (`ip link`).
- Check Wi‑Fi 6/BT (`iw list`, `rfkill list`).
- Confirm HDMI output + audio.
- Confirm panfrost for Mali‑G52 (kernel logs + `glxinfo`/`kmscube`).

Armbian Rockchip kernel tree and panfrost enablement (for reference):

- https://github.com/armbian/linux-rockchip  
- https://github.com/armbian/linux-rockchip/pull/249

---

## 4. LXC on Debian/Armbian Host

### 4.1 LXC docs

- LXC documentation hub:  
  https://linuxcontainers.org/lxc/documentation/

- LXC homepage:  
  https://linuxcontainers.org

- LXC man page (config reference):  
  https://man7.org/linux/man-pages/man7/lxc.7.html

- Debian‑oriented guide (unprivileged containers and bridges, good patterns):  
  https://blog.michaelkelly.org/2023/09/lxc-containers-on-debian-part-1-setup/

### 4.2 Install and basic check

```bash
sudo apt update
sudo apt install lxc bridge-utils
sudo lxc-checkconfig
```

You want:

- Namespaces, cgroup v2, veth, bridge, etc. all **enabled**.

For OpenWrt specifically, use a **privileged container**, as that matches most working how‑tos and reduces surprises.

---

## 5. OpenWrt in LXC

### 5.1 OpenWrt LXC documentation

Primary guide:

- OpenWrt in LXC containers:  
  https://openwrt.org/docs/guide-user/virtualization/lxc

It documents two approaches:

- Using `lxc-create -t download` with an OpenWrt image where available.
- Using a manually downloaded rootfs tarball and custom config.

### 5.2 Creating the OpenWrt container

#### Option A: via `download` template (if OpenWrt appears)

```bash
sudo lxc-create -n openwrt -t download -- -d openwrt -a arm64
```

Choose a suitable release when prompted (if arm64+OpenWrt is offered by the template).

#### Option B: via rootfs extraction (manual)

Follow the **“Via rootfs extraction”** section in the OpenWrt LXC doc:

1. Create container dir:

   ```bash
   sudo mkdir -p /var/lib/lxc/openwrt/rootfs
   cd /var/lib/lxc/openwrt
   ```

2. Download an OpenWrt aarch64 rootfs tarball from the OpenWrt download site.

3. Extract into `rootfs/`:

   ```bash
   sudo tar xvf openwrt-*-rootfs.tar.gz -C rootfs
   ```

4. Create `/var/lib/lxc/openwrt/config` (see next subsection).

For more automation ideas and caveats (x86/Proxmox oriented, but pattern is useful):

- OpenWrt 23.05 LXC container (Proxmox) – GitHub Gist:  
  https://gist.github.com/suuhm/053f819b000bee4af922d66ff6c5d32e

- “Installing OpenWRT on top of LXC in Proxmox” (discussion and networking examples):  
  https://lowendspirit.com/discussion/2266/installing-openwrt-on-top-of-lxc-in-proxmox

### 5.3 Example `config` for OpenWrt‑LXC (WAN + LAN)

Minimal example (adapt to your paths / MACs):

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

Additional OpenWrt‑LXC networking discussion:

- “How to configure OpenWrt networking with LXC?” (Reddit):  
  https://www.reddit.com/r/openwrt/comments/1engfcz/how_to_configure_openwrt_networking_with_lxc/

---

## 6. Host Networking (Bridges Only)

Goal: host is just a **bridge + hypervisor**; OpenWrt does all routing.

### 6.1 Bridging physical NICs

Assume:

- `eth0` = physical WAN port
- `eth1` = physical LAN port

Create two Linux bridges (you can use systemd‑networkd, NetworkManager, or `ip` commands).

Example with systemd‑networkd style (conceptual config):

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

# Optional: Host management IP on br-lan for SSH
# /etc/systemd/network/30-br-lan.network
[Match]
Name=br-lan

[Network]
Address=192.168.1.2/24
Gateway=192.168.1.1
```

Notes:

- **No IP on `br-wan`** → full pass‑through to the OpenWrt WAN interface.
- Optional static IP on `br-lan` → host is reachable for SSH even if OpenWrt is down.
- LXC veth interfaces are attached via `lxc.net.0.link = br-wan` and `lxc.net.1.link = br-lan`.

---

## 7. Weston (DRM/KMS) on Host

### 7.1 Docs

- Weston docs:  
  https://wayland.pages.freedesktop.org/weston/index.html

- Jetson Weston guide (good DRM/KMS + kiosk overview – conceptually applicable):  
  https://docs.nvidia.com/jetson/archives/r38.4/DeveloperGuide/SD/WindowingSystems/WestonWayland.html

- Weston repo/mirror:  
  https://github.com/intel/Intel-Distribution-of-Weston

### 7.2 Install Weston

```bash
sudo apt install weston
```

Create a dedicated user:

```bash
sudo adduser tvuser
```

Configure **autologin** on tty1 (via `getty@tty1.service` override) for `tvuser`, then use a user‑level systemd unit to start Weston.

### 7.3 Example systemd user unit for Weston

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

- Waydroid: install on desktops:  
  https://docs.waydro.id/usage/install-on-desktops

- GitHub doc (same content source):  
  https://github.com/waydroid/docs/blob/master/usage/install-on-desktops.md

On Debian/Armbian:

```bash
curl -s https://repo.waydro.id | sudo bash      # add repo (use -s bookworm/trixie if needed)
sudo apt install waydroid
sudo waydroid init
```

### 8.2 Swap in Android TV (Waydroid‑ATV)

Reference release (example – use latest):

- Waydroid‑ATV builds with install notes:  
  https://newreleases.io/project/github/WayDroid-ATV/waydroid-androidtv-builds

From a typical release (e.g. lineages 20 ATV):

1. Download `lineage-20.0-*-UNOFFICIAL-WaydroidATV_*.zip`.
2. Extract `system.img` and `vendor.img`.
3. Place them into Waydroid extra images directory:

   ```bash
   sudo mkdir -p /etc/waydroid-extra/images/
   sudo cp system.img /etc/waydroid-extra/images/system.img
   sudo cp vendor.img /etc/waydroid-extra/images/vendor.img
   ```

4. Re‑init Waydroid:

   ```bash
   sudo waydroid init -f
   ```

Background context:

- Reddit thread announcing Waydroid‑ATV effort and linking to the repo:  
  https://www.reddit.com/r/waydroid/comments/1emc5kv/finally_got_android_tv_compiled_for_waydroid/

### 8.3 Autostart Waydroid full UI under Weston

Once Weston runs as `tvuser`:

```bash
sudo waydroid container start      # usually root; handle via sudoers or helper
waydroid show-full-ui              # run as tvuser in Weston session
```

Wrap that in a script and a systemd user unit.

**Script:**

```bash
# /usr/local/bin/waydroid-atv-start.sh
#!/bin/bash
set -e

# Ensure container is up
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

On boot, `tvuser` autologins → Weston starts → Waydroid‑ATV launches full‑screen.

---

## 9. OpenWrt Configuration Inside the Container

Once the container is running:

```bash
sudo lxc-start -n openwrt -F
# or
sudo lxc-attach -n openwrt
```

Inside OpenWrt:

1. In `/etc/config/network`:
   - Configure `eth0` as `wan` (DHCP client or static to upstream).  
   - Configure `eth1` as `lan` (static, e.g. 192.168.1.1/24, plus DHCP server).

2. Configure firewall, DNS, and optional Wi‑Fi AP as usual for an OpenWrt router.

Reference again:

- OpenWrt in LXC containers:  
  https://openwrt.org/docs/guide-user/virtualization/lxc

---

## 10. Boot Flow Summary

1. **Bootloader → Armbian/Debian on RK3576**.
2. systemd:
   - Brings up `br-wan`, `br-lan`, and physical NICs.  
   - Starts the **OpenWrt** LXC container (router).  
3. On tty1, `tvuser` autologin:
   - `systemd --user` starts **Weston** (DRM/KMS).  
   - Weston then starts **Waydroid‑ATV** full‑screen.

The BPI‑M5 Pro now behaves as a travel router with a full Android TV front‑end on HDMI, all in one device.
