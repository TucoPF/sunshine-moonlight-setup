# 🎮 Headless Linux VM Gaming & Workstation Streaming Guide
## Low-Latency Sunshine/Moonlight Setup on Proxmox with AMD iGPU Passthrough

This repository provides an exhaustive, step-by-step engineering guide to deploying a headless Linux Virtual Machine (Debian 13 Trixie) on Proxmox VE, optimized for ultra-low latency desktop and game streaming via **Sunshine** to a **Moonlight** client (Intel NUC).

It covers host network tuning, VM operating system cleanup, GPU passthrough configurations, Wayland/KDE desktop optimization, and client-side performance adjustments.

---

## 📐 Architecture & Topology

```mermaid
graph TD
    subgraph Host ["skynet - Proxmox Host (Ryzen 9700X)"]
        VM[Debian 13 VM Workstation]
        iGPU[AMD Radeon iGPU Passthrough] --> VM
        Plug[HDMI Dummy Plug 1080p] --> iGPU
    end
    
    subgraph Network ["Local Network (BBR & fq Optimized)"]
        VM -- "Sunshine (KMS Video + PipeWire Audio)" --> NUC[Intel NUC Client (Debian 13)]
    end

    NUC -- "Moonlight-qt (Intel HD 620 HW Decode)" --> Display["TV Monitor 1368x768 / 1080p"]
```

*   **Host Server (`skynet`)**: Custom ATX Tower, AMD Ryzen 7 9700X, 32GB DDR5, running Proxmox VE 9.2.3. Passes the onboard graphics (`79:00.0` VGA and `79:00.1` Audio) to the guest VM.
*   **Streaming Guest VM (`Workstation` - `fddf::2222` / `192.168.1.22`)**: Debian 13 (Trixie), minimal KDE Plasma Wayland session, GPU-accelerated encoding via VA-API/Vulkan.
*   **Client Node (`Nuc` - `fddf::3` / `192.168.1.253`)**: Intel NUC7i3DNHE (i3-7100U, HD 620 GPU), running Moonlight-qt client over Wi-Fi.

---

## 🎛️ Step 1: Host Network Optimization (`matrix` & `skynet`)

To support high-bitrate video streaming and fast storage backups without introducing packet queueing delays (bufferbloat), TCP BBR congestion control and Fair Queueing are enabled on the Proxmox hosts.

1.  Consolidate and create `/etc/sysctl.d/99-homelab-network.conf` on both hosts:
    ```text
    # --- Network Buffer Optimizations (25MB Max) ---
    net.core.rmem_max = 26214400
    net.core.rmem_default = 26214400
    net.core.wmem_max = 26214400
    net.core.wmem_default = 26214400

    # --- TCP BBR Congestion Control & Pacing ---
    net.core.default_qdisc = fq
    net.ipv4.tcp_congestion_control = bbr

    # --- IPv6 SLAAC Router Advertisement ---
    net.ipv6.conf.all.accept_ra = 2
    net.ipv6.conf.default.accept_ra = 2
    net.ipv6.conf.vmbr0.accept_ra = 2
    ```
2.  Clean up any legacy files (`99-pve-firewall.conf`, `99-pve-ipv6.conf`) and apply the new configuration:
    ```bash
    sudo rm -f /etc/sysctl.d/99-pve-firewall.conf /etc/sysctl.d/99-pve-ipv6.conf
    sudo sysctl --system
    ```

---

## 🏗️ Step 2: Guest VM OS Cleanup & Validation

To achieve a clean, warning-free system boot and prevent initialization hangs, we clean up the VM boot sequence.

### A. Disable Virtual Hotplug Modules
Debian VMs under QEMU often log repetitive errors regarding virtual PCI hotplug failures.
1.  Disable the `shpchp` module loading via a dummy installation redirection:
    ```bash
    echo "install shpchp /bin/true" | sudo tee /etc/modprobe.d/blacklist-shpchp.conf
    ```

### B. Disable systemd SSH Socket Generator
If the VM lacks a virtual socket configuration, it will log errors at boot.
1.  Mask the generator by linking it to `/dev/null`:
    ```bash
    sudo mkdir -p /etc/systemd/system-generators
    sudo ln -sf /dev/null /etc/systemd/system-generators/systemd-ssh-generator
    ```

### C. Repair EFI FAT volume Dirty Bit
If the VM was shut down uncleanly, the EFI system partition logs filesystem dirtiness warnings at boot.
1.  Install DOS filesystem tools and scan the partition:
    ```bash
    sudo apt install -y dosfstools
    sudo fsck.vfat -a /dev/sda1
    ```

### D. Update Initramfs
Apply the modules configuration modifications to the boot ramdisk:
```bash
sudo update-initramfs -u -k all
```

---

## 🖥️ Step 3: Graphical Environment & Autologin Setup

For Sunshine to capture the display, a valid Wayland graphical session must run automatically on boot. We configure a minimal KDE Plasma desktop running on SDDM with passwordless autologin.

1.  **Install Minimal Desktop & Audio Stack**:
    ```bash
    sudo apt install -y --no-install-recommends \
      kde-plasma-desktop sddm xwayland \
      pipewire pipewire-audio wireplumber \
      xdg-desktop-portal-kde mesa-va-drivers mesa-vulkan-drivers
    ```
2.  **Configure SDDM Autologin**:
    Create `/etc/sddm.conf.d/autologin.conf` to automatically log in the user `tuco` directly into the Wayland session:
    ```ini
    [Autologin]
    User=tuco
    Session=plasma
    ```
3.  **Add User to Input Group**:
    Allow the streaming user to emulate mouse and keyboard events:
    ```bash
    sudo usermod -a -G input tuco
    ```

---

## 🎮 Step 4: Sunshine Installation & Hardening

1.  **Download and Install Sunshine**:
    ```bash
    wget -O /tmp/sunshine.deb https://github.com/LizardByte/Sunshine/releases/latest/download/sunshine-debian-trixie-amd64.deb
    sudo apt install -y /tmp/sunshine.deb
    ```
2.  **Apply Capabilities for Low-Latency Real-Time Capturing**:
    Enable raw KMS screencasting and high scheduling priority without executing Sunshine as `root`:
    ```bash
    sudo setcap cap_sys_admin,cap_sys_nice+p $(readlink -f $(which sunshine))
    ```
3.  **Configure Sunshine (`/home/tuco/.config/sunshine/sunshine.conf`)**:
    ```text
    capture = kms
    min_log_level = info
    address_family = both
    origin_web_ui_allowed = lan
    csrf_allowed_origins = https://[fddf::2222],https://192.168.1.22
    ```
    *Note: Replace ULA `fddf::2222` and local IP `192.168.1.22` with your VM's network parameters.*
4.  **Auto-Discovery (Avahi Setup)**:
    Install Avahi so Moonlight clients can automatically discover the VM:
    ```bash
    sudo apt install -y avahi-daemon
    sudo systemctl enable --now avahi-daemon
    ```
5.  **Enable the Sunshine systemd User Service**:
    ```bash
    systemctl --user enable --now app-dev.lizardbyte.app.Sunshine.service
    ```

---

## 💻 Step 5: NUC Client Latency Optimization

To eliminate client-side video decoding lags and network jitter on the Intel NUC client (NUC7i3DNHE, Intel Core i3-7100U, HD Graphics 630) running Debian 13 Trixie (Headless):

### A. Disable Wi-Fi Power Saving
Ensure the wireless interface (`wlp1s0`) never enters low-power sleep states during streaming:
```bash
sudo iw dev wlp1s0 set power_save off
```

### B. Configure CPU Governor to Performance Mode
Force all Intel CPU cores to react instantly to incoming frames without stepping frequency. Create a persistent systemd service at `/etc/systemd/system/cpu-governor.service`:

```ini
[Unit]
Description=Set CPU governor to performance
After=sysfs.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'for file in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do echo performance > "$file"; done'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable and start the service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now cpu-governor.service
```

### C. Static Network Configuration (`/etc/network/interfaces`)
To guarantee persistent, conflict-free connectivity on boot, configure `/etc/network/interfaces` for static dual-stack IPv4/IPv6 using the native `ifupdown` tool. Ensure all SSID/password fields with special characters (like `*`) are explicitly quoted to prevent parsing failures:

```text
source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# Primary Wi-Fi Interface configuration
auto wlp1s0

# IPv4 Static Configuration
iface wlp1s0 inet static
        address 192.168.1.253
        netmask 255.255.255.0
        gateway 192.168.1.1
        dns-nameservers 1.1.1.1 9.9.9.9
        wpa-ssid "Maison"
        wpa-psk  "Eliculolaop38*"

# IPv6 Static Configuration (Link-Local Gateway scoped to interface)
iface wlp1s0 inet6 static
        address fddf::3
        netmask 64
        gateway fe80::3a07:16ff:fe21:c7c4%wlp1s0
        dns-nameservers 2606:4700:4700::1111 2001:4860:4860::8888
```

*Note: Restarting the network or transitioning from dynamic leases requires killing legacy DHCP processes and flushing the interface to apply the static parameters cleanly:*
```bash
sudo pkill -9 dhclient
sudo ip addr flush dev wlp1s0
sudo systemctl restart networking
```

### D. Headless EGLFS Direct Display Setup (No Xorg/Wayland Compositor)
Rather than executing a heavy desktop environment (like GNOME/KDE) or even a lightweight Wayland Kiosk (like `cage`), launch Moonlight directly on the GPU using the native **EGLFS (KMS/DRM via Mesa)** platform plugin. This yields direct-to-scanout rendering, bypassing any compositor buffering and saving RAM.

1.  **Install Flatpak & Configure Flathub (User Space)**:
    ```bash
    sudo apt update && sudo apt install -y flatpak
    flatpak remote-add --user --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
    source /etc/profile.d/flatpak.sh
    flatpak install --user -y flathub com.moonlight_stream.Moonlight
    ```
2.  **Execute Directly from TTY/SSH**:
    ```bash
    flatpak run com.moonlight_stream.Moonlight
    ```
    *Note: Moonlight will auto-detect the lack of X11/Wayland display server and transparently initialize the EGLFS plugin directly on `/dev/dri/card0`.*

### E. Input Device & Latency Tuning
USB polling rate issues and mouse acceleration curves can heavily degrade the responsiveness of the remote cursor.

*   **USB Polling Rate Bottleneck (Logitech G305)**: Gaming wireless dongles run at 1000 Hz by default, flooding the client CPU with USB interrupt requests. On low-power hardware, this creates severe lag. Toggle the physical button behind the G305 scroll wheel to **Green LED (Endurance Mode / 125 Hz)** to drastically reduce CPU overhead. Use **Orange LED (1600 DPI)** for standard cursor sensitivity.
*   **Mouse Acceleration Settings (KDE Guest)**: Force a Flat acceleration curve on the guest virtual pointer. Create or edit `/home/tuco/.config/kcminputrc` to include:
    ```ini
    [Libinput][48879][57005][Mouse passthrough]
    Enabled=true
    PointerAccelerationProfile=1
    PointerAcceleration=0.000
    ```
    *(Note: KDE System Settings may visually default the dropdown back to "QEMU Tablet" due to a GUI bug, but Wayland respects the underlying file values).*

### F. Moonlight Client Profile Settings
Open Moonlight-qt and configure:
*   **Resolution**: Match TV/Monitor native resolution (`1368x768` or `1920x1080`) to prevent video downscaling and scaling overhead.
*   **Video Decoder**: Force **Hardware Decoding** (utilizes the Intel HD 630 VA-API block).
*   **V-Sync**: **Disable V-Sync** (reduces input lag by 1 full frame, accepting minor screen tearing in exchange for immediate cursor response).
*   **Frame Pacing**: Select **Lowest Latency** (Block-based pacing).
