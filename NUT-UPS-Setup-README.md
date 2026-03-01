# NUT UPS Setup — Server Rack Documentation

This document covers the full setup of Network UPS Tools (NUT) for a MinuteMan UPS, including the NUT server on a Raspberry Pi, NUT clients on all machines, Home Assistant integration, and Proxmox shutdown/startup ordering.

---

## Table of Contents

1. [Hardware Overview](#hardware-overview)
2. [Shutdown Strategy](#shutdown-strategy)
3. [NUT Server — Raspberry Pi](#nut-server--raspberry-pi)
4. [NUT Clients — Debian LXCs](#nut-clients--debian-lxcs)
5. [NUT Clients — TrueNAS SCALE](#nut-clients--truenas-scale)
6. [NUT Client — Proxmox](#nut-client--proxmox)
7. [NUT Client — Windows (WinNUT)](#nut-client--windows-winnut)
8. [Home Assistant Integration](#home-assistant-integration)
9. [Home Assistant Dashboard](#home-assistant-dashboard)
10. [Proxmox VM/LXC Startup & Shutdown Order](#proxmox-vmlxc-startup--shutdown-order)
11. [OPNsense Firewall Rule](#opnsense-firewall-rule)

---

## Hardware Overview

**UPS:** MinuteMan Pro 1000 (1000VA / 700W)
**NUT Server:** Raspberry Pi 3B running Raspberry Pi OS
**Connection:** USB (UPS reports as Delta Electronics USB HID device)

**Devices on UPS:**
- Main server (Proxmox host with TrueNAS VM, LXCs, and VMs)
- Backup server (TrueNAS SCALE — also runs Proxmox Backup Server as a container)
- Raspberry Pi (NUT server)
- Router (OPNsense)
- Brocade switch (PoE AP)

> **Note:** The router and switch have no NUT client installed. They are intentionally left running until the battery is fully depleted to keep network communications alive between all NUT clients for as long as possible.

---

## Shutdown Strategy

NUT clients use **runtime remaining** as the shutdown trigger (more reliable than battery % on this UPS model). The UPS runtime estimate was validated during a live test.

| Stage | Machines | Runtime Trigger | Reasoning |
|-------|----------|-----------------|-----------|
| 1 | NFS/SMB client LXCs, qbit2 (Windows VM), Backup TrueNAS | 15 min (900s) | Unmount shares before TrueNAS goes down |
| 2 | Main TrueNAS VM | 10 min (600s) | Shut down after dependents are clear |
| 3 | Proxmox host + Raspberry Pi | 5 min (300s) | Last machines off before network gear loses power |

> **Important:** Treat 50% battery as effectively dead on this UPS — percentage readings are unreliable. Runtime remaining is the trustworthy metric.

---

## NUT Server — Raspberry Pi

> **Credit:** This section is based on Jeff Geerling's guide: https://www.jeffgeerling.com/blog/2025/nut-on-my-pi-so-my-servers-dont-die/
> Follow Jeff's guide through the "Confirm NUT works" step. Key commands and config snippets are reproduced below for completeness.

### 1. Install NUT

```bash
sudo apt install -y nut
```

### 2. Identify the UPS

Plug in the UPS via USB, then run:

```bash
lsusb
```

Look for a Delta Electronics entry (MinuteMan rebrands from Delta). Note the VID:PID values.

```bash
sudo nut-scanner -U
```

This will suggest a config block for your UPS.

### 3. Configure USB Permissions

Create a udev rule so NUT can access the USB device:

```bash
sudo nano /etc/udev/rules.d/99-nut-ups.rules
```

Add (replace VID with your actual vendor ID from lsusb, e.g. `05dd`):

```
SUBSYSTEM=="usb", ATTR{idVendor}=="<your-vendor-id>", MODE="0664", GROUP="nut"
```

Reload udev:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Add the nut user to the dialout group:

```bash
sudo usermod -aG dialout nut
```

### 4. `/etc/nut/nut.conf`

```ini
MODE=netserver
```

### 5. `/etc/nut/ups.conf`

```ini
[minuteman]
  driver = usbhid-ups
  port = auto
  vendorid = <your-vendor-id>
  productid = <your-product-id>
  desc = "MinuteMan UPS"
  override.battery.runtime.low = 900
```

### 6. `/etc/nut/upsd.conf`

```ini
LISTEN 127.0.0.1 3493
LISTEN 0.0.0.0 3493
```

### 7. `/etc/nut/upsd.users`

```ini
[admin]
  password = <admin-password>
  upsmon master

[observer]
  password = <observer-password>
  upsmon slave
```

> **Security:** Use strong unique passwords. The `observer` user is used by all remote NUT clients. The `admin` user is for the Pi itself.

### 8. `/etc/nut/upsmon.conf`

Find and update these two lines in the existing file (do not replace the whole file):

```ini
MONITOR minuteman@localhost 1 admin <admin-password> primary
FINALDELAY 180
```

> `FINALDELAY 180` gives all client machines 3 minutes to finish shutting down before the Pi powers off and the UPS cuts its load.

### 9. Start NUT

```bash
sudo systemctl enable nut-server nut-driver nut-monitor
sudo systemctl start nut-server nut-driver nut-monitor
```

### 10. Confirm NUT Works

```bash
upsc minuteman@localhost
```

You should see a full list of UPS variables. Verify `ups.status` shows `OL` (on line).

---

## NUT Clients — Debian LXCs

**Applies to:** sonarr, radarr, qbittorrent, immich, and any other Debian-based LXC that mounts a TrueNAS share.

**Shutdown threshold:** 15 minutes runtime remaining (Stage 1)

### 1. Install NUT Client

```bash
sudo apt update && sudo apt install nut-client -y
```

### 2. `/etc/nut/nut.conf`

```bash
echo "MODE=netclient" | sudo tee /etc/nut/nut.conf
```

### 3. `/etc/nut/upsmon.conf`

Replace the entire file contents with:

```ini
MONITOR minuteman@<pi-ip> 1 observer <observer-password> slave
MINSUPPLIES 1
SHUTDOWNCMD "/sbin/shutdown -h now"
POLLFREQ 5
POLLFREQALERT 5
HOSTSYNC 15
DEADTIME 15
RBWARNTIME 43200
NOCOMMWARNTIME 300
FINALDELAY 5
BATTERY_RUNTIME_LOW 900
```

### 4. Enable and Start

```bash
sudo systemctl enable nut-monitor
sudo systemctl start nut-monitor
sudo systemctl status nut-monitor
```

Confirm you see `Active: active (running)`. Then verify connection:

```bash
sudo tail -f /var/log/syslog | grep -i ups
```

You should see `on line power` within 10-15 seconds.

---

## NUT Clients — TrueNAS SCALE

TrueNAS SCALE uses an immutable OS image that gets replaced on updates. NUT config must be written via an **Init/Shutdown Script** stored on the data pool to survive updates.

### Backup TrueNAS (Stage 1 — 15 min / 900s)

#### Step 1: SSH into Backup TrueNAS and create the script

```bash
mkdir -p /mnt/<your-pool-name>/configs
nano /mnt/<your-pool-name>/configs/nut-setup.sh
```

Paste:

```bash
#!/bin/bash
echo "MODE=netclient" > /etc/nut/nut.conf

cat > /etc/nut/upsmon.conf << 'EOF'
MONITOR minuteman@<pi-ip> 1 observer <observer-password> slave
MINSUPPLIES 1
SHUTDOWNCMD "/sbin/shutdown -h now"
POLLFREQ 5
POLLFREQALERT 5
HOSTSYNC 15
DEADTIME 15
RBWARNTIME 43200
NOCOMMWARNTIME 300
FINALDELAY 5
BATTERY_RUNTIME_LOW 900
EOF

systemctl restart nut-monitor
```

Make it executable:

```bash
chmod +x /mnt/<your-pool-name>/configs/nut-setup.sh
```

#### Step 2: Register as Init Script in TrueNAS UI

Go to **System → Advanced → Init/Shutdown Scripts → Add**:

- **Description:** NUT Client Setup
- **Type:** Script
- **Script:** `/mnt/<your-pool-name>/configs/nut-setup.sh`
- **When:** Post Init
- **Enabled:** ✓

> **Note:** Disable the built-in TrueNAS NUT service under **System → NUT** to avoid conflicts.

---

### Main TrueNAS VM (Stage 2 — 10 min / 600s)

Same process as Backup TrueNAS but with a different runtime threshold:

```bash
#!/bin/bash
echo "MODE=netclient" > /etc/nut/nut.conf

cat > /etc/nut/upsmon.conf << 'EOF'
MONITOR minuteman@<pi-ip> 1 observer <observer-password> slave
MINSUPPLIES 1
SHUTDOWNCMD "/sbin/shutdown -h now"
POLLFREQ 5
POLLFREQALERT 5
HOSTSYNC 15
DEADTIME 15
RBWARNTIME 43200
NOCOMMWARNTIME 300
FINALDELAY 5
BATTERY_RUNTIME_LOW 600
EOF

systemctl restart nut-monitor
```

Register in TrueNAS UI the same way under **System → Advanced → Init/Shutdown Scripts**.

> **Note:** Proxmox Backup Server running as a TrueNAS SCALE app/container does **not** need its own NUT client. TrueNAS SCALE automatically stops all running apps gracefully as part of its own shutdown sequence.

---

## NUT Client — Proxmox

Proxmox is Debian-based and config files in `/etc` survive `apt` updates. If `apt` asks whether to keep your config file during a NUT package update, answer **N** (keep existing).

**Shutdown threshold:** 5 minutes runtime remaining (Stage 3)

### 1. Install NUT Client

```bash
sudo apt update && sudo apt install nut-client -y
```

### 2. `/etc/nut/nut.conf`

```bash
echo "MODE=netclient" | sudo tee /etc/nut/nut.conf
```

### 3. `/etc/nut/upsmon.conf`

Replace the entire file with:

```ini
MONITOR minuteman@<pi-ip> 1 observer <observer-password> slave
MINSUPPLIES 1
SHUTDOWNCMD "/sbin/shutdown -h now"
POLLFREQ 5
POLLFREQALERT 5
HOSTSYNC 15
DEADTIME 15
RBWARNTIME 43200
NOCOMMWARNTIME 300
FINALDELAY 5
BATTERY_RUNTIME_LOW 300
```

### 4. Enable and Start

```bash
sudo systemctl enable nut-monitor
sudo systemctl start nut-monitor
sudo systemctl status nut-monitor
```

### 5. Protect Config from Unattended Upgrades

```bash
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

Add inside the `Dpkg::Options` section:

```
Dpkg::Options:: "--force-confold";
```

---

## NUT Client — Windows (WinNUT)

**Applies to:** qbit2 Windows VM (Stage 1 — 15 min / 900s)

### 1. Install WinNUT

Download and install WinNUT from: https://github.com/gawindx/WinNUT-Client

### 2. Configure

In the WinNUT configuration UI set:

- **UPS Host:** `<pi-ip>`
- **Port:** `3493`
- **UPS Name:** `minuteman`
- **Username:** `observer`
- **Password:** `<observer-password>`
- **Shutdown threshold:** 15 minutes runtime remaining

> WinNUT handles the Windows shutdown command internally — no manual `SHUTDOWNCMD` editing required.

---

## Home Assistant Integration

Home Assistant connects to NUT as a **slave client** over the network.

### Prerequisites

- HA and the Pi must be able to reach each other on port `3493`
- If on different VLANs, add an OPNsense firewall rule (see [OPNsense Firewall Rule](#opnsense-firewall-rule))

### Add the Integration

Go to **Settings → Devices & Services → Add Integration → Network UPS Tool (NUT)**:

- **Host:** `<pi-ip>`
- **Port:** `3493`
- **Username:** `observer`
- **Password:** `<observer-password>`

### Enable Additional Entities

By default HA only enables a few NUT sensors. Go to **Settings → Devices & Services → NUT → your UPS device** and enable:

- `sensor.minuteman_load`
- `sensor.minuteman_battery_runtime`
- `sensor.minuteman_battery_voltage`
- `sensor.minuteman_input_voltage`
- `sensor.minuteman_output_voltage`
- `sensor.minuteman_nominal_power`
- `sensor.minuteman_status_data`

### Template Sensors (`configuration.yaml`)

Add the following to `/config/configuration.yaml` on your HA instance. Access via **Settings → Add-ons → File Editor** or SSH.

```yaml
template:
  - sensor:
      - name: "Minuteman Output Wattage"
        unique_id: minuteman_output_wattage
        unit_of_measurement: "W"
        state_class: measurement
        device_class: power
        state: "{{ (states('sensor.minuteman_load') | float(0) / 100 * 700) | round(1) }}"

      - name: "Minuteman Runtime Formatted"
        unique_id: minuteman_runtime_formatted
        state: >
          {% set seconds = states('sensor.minuteman_battery_runtime') | int(0) %}
          {% set m = (seconds // 60) %}
          {% set s = (seconds % 60) %}
          {{ m }}m {{ s }}s

      - name: "Minuteman Instantaneous Cost"
        unique_id: minuteman_instantaneous_cost
        unit_of_measurement: "$/hr"
        state: "{{ (states('sensor.minuteman_output_wattage') | float(0) / 1000 * 0.13) | round(4) }}"

      - name: "Minuteman 24h Cost"
        unique_id: minuteman_24h_cost
        unit_of_measurement: "$"
        state: "{{ (states('sensor.minuteman_energy_24h') | float(0) * 0.13) | round(4) }}"

sensor:
  - platform: integration
    source: sensor.minuteman_output_wattage
    name: minuteman_energy_kwh
    unit_prefix: k
    round: 3
    method: trapezoidal

utility_meter:
  minuteman_energy_24h:
    source: sensor.minuteman_energy_kwh
    cycle: daily
```

> **Note:** Update `0.13` to your actual cost per kWh. Update `700` if your UPS wattage rating differs.
> The 24h cost sensor starts at $0.00 on first setup and resets at midnight each day.

After saving, go to **Developer Tools → YAML → Check Configuration** then restart HA.

---

## Home Assistant Dashboard

This dashboard requires the **card-mod** custom component installed via HACS.

### Install card-mod

1. Go to **HACS** in the HA sidebar
2. Search for **card-mod**
3. Click **Download**
4. Restart HA via **Developer Tools → YAML → Restart**

### Create the Dashboard

1. Go to **Settings → Dashboards → Add Dashboard**
2. Name it `UPS`, choose an icon, click **Create**
3. Navigate to the new dashboard
4. Click the **pencil icon** → **three dot menu** → **Edit in YAML**
5. Replace all contents with the YAML below and click **Save**

> **Note:** This dashboard uses the **Masonry** view type. If you created the dashboard with the Sections layout, change it via the pencil icon next to the view name → **View type → Masonry**.

```yaml
views:
  - title: UPS
    path: ups
    cards:
      - type: grid
        columns: 3
        cards:
          - type: gauge
            entity: sensor.minuteman_battery_charge
            name: Battery Charge
            unit: "%"
            min: 0
            max: 100
            severity:
              green: 50
              yellow: 20
              red: 0
          - type: gauge
            entity: sensor.minuteman_load
            name: UPS Load
            unit: "%"
            min: 0
            max: 100
            severity:
              green: 0
              yellow: 70
              red: 90
          - type: gauge
            entity: sensor.minuteman_output_wattage
            name: Output Watts
            min: 0
            max: 700
            severity:
              green: 0
              yellow: 500
              red: 620

      - type: grid
        columns: 2
        cards:
          - type: entity
            entity: sensor.minuteman_status
            name: Power Source
            icon: mdi:power-plug
            card_mod:
              style: |
                ha-card {
                  background-color: {{ 'rgba(0, 180, 0, 0.3)' if is_state('sensor.minuteman_status', 'Online') else 'rgba(180, 0, 0, 0.3)' }};
                  border: 2px solid {{ 'rgba(0, 180, 0, 0.8)' if is_state('sensor.minuteman_status', 'Online') else 'rgba(180, 0, 0, 0.8)' }};
                }
          - type: entity
            entity: sensor.minuteman_runtime_formatted
            name: Runtime Remaining
            icon: mdi:timer-outline

      - type: grid
        columns: 2
        cards:
          - type: entity
            entity: sensor.minuteman_instantaneous_cost
            name: Current Cost Rate
            icon: mdi:currency-usd
          - type: entity
            entity: sensor.minuteman_24h_cost
            name: Cost Today
            icon: mdi:cash-clock

      - type: grid
        columns: 1
        cards:
          - type: entities
            title: UPS Information
            entities:
              - entity: sensor.minuteman_nominal_power
                name: Rated Capacity
              - entity: sensor.minuteman_status
                name: Status
              - entity: sensor.minuteman_status_data
                name: Raw Status
              - entity: sensor.minuteman_battery_voltage
                name: Battery Voltage

      - type: grid
        columns: 1
        cards:
          - type: history-graph
            title: Input Voltage (8h)
            hours_to_show: 8
            entities:
              - entity: sensor.minuteman_input_voltage
                name: Input Voltage

      - type: grid
        columns: 1
        cards:
          - type: history-graph
            title: Output Voltage (8h)
            hours_to_show: 8
            entities:
              - entity: sensor.minuteman_output_voltage
                name: Output Voltage

      - type: grid
        columns: 1
        cards:
          - type: history-graph
            title: Output Wattage (8h)
            hours_to_show: 8
            entities:
              - entity: sensor.minuteman_output_wattage
                name: Output Watts

      - type: grid
        columns: 1
        cards:
          - type: history-graph
            title: Load % (8h)
            hours_to_show: 8
            entities:
              - entity: sensor.minuteman_load
                name: Load
```

---

## Proxmox VM/LXC Startup & Shutdown Order

Proxmox starts VMs/LXCs in **ascending** order and shuts them down in **descending** order.

The NUT clients on Stage 1 and Stage 2 machines handle their own shutdown **before** Proxmox begins its shutdown sequence. The Proxmox ordering below is therefore optimized primarily for **startup order** (ensuring dependencies are ready before dependents), with NUT providing the safety net for correct shutdown ordering.

To set these values: click each VM/LXC in Proxmox → **Options** → **Start/Shutdown Order**

| Order | ID | Name | Type | Startup Delay | Shutdown Timeout | NUT Client |
|-------|----|------|------|---------------|-----------------|------------|
| 10 | 101 | postgresql | LXC | 0s | 20s | No (Proxmox handles) |
| 20 | 102 | uptimekuma | LXC | 45s | 20s | No (Proxmox handles) |
| 20 | 1600 | proxy/NPM | LXC | 45s | 20s | No (Proxmox handles) |
| 30 | 103 | ubuntu | LXC | 90s | 20s | No (Proxmox handles) |
| 30 | 1000 | homeassistant | VM | 90s | 60s | No (Proxmox handles) |
| 40 | 1200 | truenas | VM | 135s | 60s | Yes — 600s (Stage 2) |
| 50 | 110 | seerr | LXC | 180s | 20s | No (Proxmox handles) |
| 50 | 120 | sonarr | LXC | 180s | 20s | Yes — 900s (Stage 1) |
| 50 | 130 | radarr | LXC | 180s | 20s | Yes — 900s (Stage 1) |
| 50 | 140 | qbittorrent | LXC | 180s | 20s | Yes — 900s (Stage 1) |
| 50 | 150 | recyclarr | LXC | 180s | 20s | No (Proxmox handles) |
| 50 | 1100 | immich | VM | 180s | 60s | Yes — 900s (Stage 1) |
| 60 | 1400 | qbit2 | VM | 225s | 60s | Yes — 900s (WinNUT) |
| 70 | 1300 | motherboard | VM | 270s | 60s | No (currently off) |

> **Startup delay note:** TrueNAS (order 40, 135s delay) must be fully booted and shares mounted before the media apps at order 50 (180s delay) start. Adjust the delay if TrueNAS takes longer than ~45 seconds to fully mount its pools on your hardware.

---

## OPNsense Firewall Rule

If Home Assistant and the Raspberry Pi NUT server are on **different VLANs**, add a firewall rule in OPNsense to allow HA to reach NUT.

### Rule Settings

Go to **Firewall → Rules → \<HA VLAN interface\>** and add:

| Field | Value |
|-------|-------|
| Action | Pass |
| Interface | HA's VLAN interface (traffic originates here) |
| Protocol | TCP |
| Source | HA's IP or subnet |
| Destination | Raspberry Pi IP |
| Destination Port | 3493 |
| Description | Allow HA to NUT server |

OPNsense is stateful by default — you only need this one rule. Return traffic is permitted automatically.

> **Also check:** If any other NUT clients (Proxmox, LXCs) are on different VLANs from the Pi, add equivalent rules for each. All NUT clients communicate with the Pi on port **3493/TCP**.

---

*Last updated: March 2026*
