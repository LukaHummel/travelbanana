# travelbanana
Travelrouter with integrated Android TV based on Banana PI BPI-M5 Pro
# BPI‑M5 Pro Travel Router + Android TV Stack (Cage + Waydroid)

This document describes how to build a **travel router with integrated Android TV** on a **Banana Pi BPI‑M5 Pro (RK3576)** using:

- **Debian/Armbian host**
- **Cage** as a Wayland kiosk compositor
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

 ├─ Cage (Wayland kiosk compositor)
 │    └─ Waydroid container
 │         └─ Android TV images (Waydroid-ATV)
 │             (HDMI output, full-screen, no other apps)
 │
 └─ LXC (host)
      └─ OpenWrt container (privileged)
           ├─ WAN: veth0 ↔ br-wan ↔ physical WAN NIC
           └─ LAN: veth1 ↔ br-lan ↔ physical LAN NIC (+ Wi-Fi AP)
```

- Host networking: **bridges only** (`br-wan`, `br-lan`).
- OpenWrt container: **full router** (NAT, DHCP, DNS, firewall, Wi‑Fi AP).
- Cage: runs a **single maximized application** (`waydroid show-full-ui`) as the only visible session.

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

For GPU/kernel reference:

- Armbian Rockchip kernel tree:  
  https://github.com/armbian/linux-rockchip

---

## 4. LXC on Debian/Armbian Host

### 4.1 LXC docs

- LXC documentation hub:  
  https://linuxcontainers.org/lxc/documentation/

- LXC man page (config reference):  
  https://man7.org/linux/man-pages/man7/lxc.7.html

- Debian‑oriented guide (unprivileged containers and bridges; good patterns even if you use a privileged OpenWrt container):  
  https://blog.michaelkelly.org/2023/09/lxc-containers-on-debian-part-1-setup/

### 4.2 Install and basic check

```bash
sudo apt update
sudo apt install lxc bridge-utils
sudo lxc-checkconfig
```

You want:

- Namespaces, cgroup v2, veth, bridge, etc. all **enabled**.

For OpenWrt specifically, use a **privileged container**; it matches most working how‑tos and avoids capability surprises.

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

Choose a suitable release when prompted (if arm64 + OpenWrt is offered by the template).

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

For more automation ideas and caveats (x86/Proxmox oriented, but patterns are useful):

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

## 7. Cage as the Wayland Kiosk Compositor

### 7.1 Cage docs

- Cage home page:  
  https://www.hjdskes.nl/projects/cage/

- Cage wiki (GitHub):  
  https://github.com/cage-kiosk/cage/wiki

- Cage man page (CLI usage, options):  
  https://manpages.ubuntu.com/manpages/noble/man1/cage.1.html

Summary:

- Cage is a **Wayland kiosk compositor** that runs a **single maximized application**, preventing interaction with anything else.
- The basic invocation is:

  ```bash
  cage application [arguments...]
  ```

### 7.2 Install Cage and Waydroid

On Debian/Armbian:

```bash
sudo apt update
sudo apt install cage waydroid
```

Waydroid install/usage docs:

- Install instructions:  
  https://docs.waydro.id/usage/install-on-desktops

Waydroid basics:

```bash
curl -s https://repo.waydro.id | sudo bash
sudo apt install waydroid
sudo waydroid init
```

---

## 8. Waydroid‑Only Session with Cage

Waydroid’s “Waydroid only sessions” FAQ includes a **Cage** example.[^waydroid-sessions]

### 8.1 Using a display manager (LightDM, GDM, SDDM, etc.)

If you have a display manager and want a **login‑screen selectable “Waydroid” session**:

1. Ensure the Waydroid container is auto‑started at boot (optional but recommended):

   ```bash
   sudo systemctl enable waydroid-container
   sudo systemctl start waydroid-container
   ```

2. Create a Wayland session file:

   ```bash
   sudo mkdir -p /usr/share/wayland-sessions
   sudo nano /usr/share/wayland-sessions/waydroid.desktop
   ```

3. Put this in `waydroid.desktop`:

   ```ini
   [Desktop Entry]
   Name=WayDroid in Cage
   Comment=Android OS in a container
   Exec=/usr/bin/cage waydroid show-full-ui
   Type=Application
   ```

4. Reboot or restart your display manager. On the login screen, pick the **“WayDroid in Cage”** session.

This will:

- Start Cage as the compositor.
- Cage will run `waydroid show-full-ui` as the single full‑screen client.

### 8.2 Headless / kiosk via systemd (no display manager)

If you don’t use a display manager and want the box to **boot straight into Android TV**:

1. Ensure `waydroid-container` is enabled:

   ```bash
   sudo systemctl enable waydroid-container
   sudo systemctl start waydroid-container
   ```

2. Create a dedicated user, e.g. `tvuser`:

   ```bash
   sudo adduser tvuser
   ```

3. Configure autologin for `tvuser` on `tty1` (e.g. via `getty@tty1.service` override).

4. As `tvuser`, create a systemd user unit:

   ```bash
   mkdir -p ~/.config/systemd/user
   nano ~/.config/systemd/user/waydroid-cage.service
   ```

5. Add:

   ```ini
   [Unit]
   Description=Waydroid in Cage (Kiosk)
   After=default.target
   Wants=default.target

   [Service]
   Type=simple
   Environment= XDG_RUNTIME_DIR=/run/user/%U
   ExecStart=/usr/bin/cage waydroid show-full-ui
   Restart=on-failure

   [Install]
   WantedBy=default.target
   ```

6. Enable and start:

   ```bash
   systemctl --user enable waydroid-cage.service
   systemctl --user start waydroid-cage.service
   ```

Now boot → autologin `tvuser` on tty1 → systemd user session starts **Cage**, which in turn runs `waydroid show-full-ui` full‑screen.

---

## 9. Waydroid‑ATV Images

To get Android TV instead of the default phone/tablet UI, use **Waydroid‑ATV builds**.

### 9.1 Install Waydroid‑ATV images

1. Pick a release from:  
   https://newreleases.io/project/github/WayDroid-ATV/waydroid-androidtv-builds

2. Download a ZIP such as:

   ```text
   lineage-20.0-*-UNOFFICIAL-WaydroidATV_*.zip
   ```

3. Extract `system.img` and `vendor.img`.

4. Place them into Waydroid’s extra images directory:

   ```bash
   sudo mkdir -p /etc/waydroid-extra/images/
   sudo cp system.img /etc/waydroid-extra/images/system.img
   sudo cp vendor.img /etc/waydroid-extra/images/vendor.img
   ```

5. Re‑init Waydroid to pick up the new images:

   ```bash
   sudo waydroid init -f
   ```

Background context:

- Waydroid command‑line options (`show-full-ui` etc.):  
  https://github.com/waydroid/docs/blob/master/usage/waydroid-command-line-options.md

- Waydroid install/usage:  
  https://docs.waydro.id/usage/install-on-desktops

---

## 10. OpenWrt Configuration Inside the Container

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

Reference:

- OpenWrt in LXC containers:  
  https://openwrt.org/docs/guide-user/virtualization/lxc

---

## 11. Boot Flow Summary

1. **Bootloader → Armbian/Debian on RK3576**.
2. systemd:
   - Brings up `br-wan`, `br-lan`, and physical NICs.  
   - Starts the **OpenWrt** LXC container (router).  
   - Starts the **Waydroid** container via `waydroid-container.service` (if enabled).
3. Either:
   - Display manager: user chooses “WayDroid in Cage” → Cage starts → `waydroid show-full-ui` full‑screen.  
   - Kiosk mode: `tvuser` autologin → user systemd starts **Cage**, which runs `waydroid show-full-ui` full‑screen.

The BPI‑M5 Pro now behaves as a travel router with a full Android TV front‑end on HDMI, all in one device, with **Cage** providing a locked‑down, single‑app Wayland session.
