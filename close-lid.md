Related note:

- [Homelab Setup Notes](/Users/travisnguyen/Documents/Coursera/homelab/set-up%20homelab.md)

When you close the **ThinkPad T450 lid**, Debian by default triggers **sleep (suspend)**.

When the laptop sleeps:

* CPU stops
* network interface stops
* Docker stops responding
* SSH session drops

Therefore the SSH connection is lost.

This is expected behaviour for laptops.

---

# Fix: disable suspend on lid close

Run on T450:

```bash id="rb9qjo"
sudo nano /etc/systemd/logind.conf
```

Find these lines:

```text id="6g78cx"
#HandleLidSwitch=suspend
#HandleLidSwitchDocked=ignore
```

Change to:

```text id="54q7n9"
HandleLidSwitch=ignore
HandleLidSwitchDocked=ignore
HandleSuspendKey=ignore
HandleHibernateKey=ignore
HandlePowerKey=poweroff
```

Save:

```text id="0axj7q"
CTRL + O
ENTER
CTRL + X
```

---

# Apply configuration

```bash id="9zq6lp"
sudo systemctl restart systemd-logind
```

---

# Test

1. ensure SSH connected from Mac
2. close lid
3. wait 30–60 seconds
4. SSH should remain connected

---

# Optional: prevent other sleep triggers

Disable automatic suspend:

```bash id="px7jv9"
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

---

# Verify no suspend configured

```bash id="n38h5l"
systemctl status sleep.target
```

Should show:

```text id="p37cny"
masked
```

---

# Recommended laptop-server settings

| setting                | reason                       |
| ---------------------- | ---------------------------- |
| lid close does nothing | keeps server online          |
| suspend disabled       | prevents unexpected downtime |
| SSH enabled            | remote management            |
| Tailscale installed    | remote access outside home   |
| auto-start docker      | service resilience           |

---

# Optional power tuning

Keep screen off but system running:

```bash id="i5qnjg"
sudo nano /etc/systemd/logind.conf
```

add:

```text id="jv8zh8"
IdleAction=ignore
```

---

# Resulting behaviour

```text id="2zzl6l"
lid closed → system continues running
ssh remains connected
docker containers continue running
n8n remains accessible
```

---

If useful, next step can include:

* prevent WiFi power saving disconnects
* configure auto-reboot after power outage
* BIOS settings for always-on behaviour
* CPU power tuning for low electricity usage

Below are **server-grade stability settings** for ThinkPad T450 running headless Debian.

Goal:

* stable 24/7 uptime
* minimal power consumption
* predictable behaviour after power loss
* reliable network connectivity

---

# 1. Prevent WiFi power-saving disconnects

Linux enables WiFi power saving by default, which can drop SSH sessions.

## Check WiFi interface

```bash id="l2lh9n"
iw dev
```

Example interface:

```text id="g6tn6p"
wlp3s0
```

## Disable power saving immediately

```bash id="4z9wgb"
sudo iw dev wlp3s0 set power_save off
```

Verify:

```bash id="vy3z5x"
iw dev wlp3s0 get power_save
```

Expected:

```text id="6fq3v8"
Power save: off
```

---

## Make persistent after reboot

Create config:

```bash id="gxz0dy"
sudo nano /etc/NetworkManager/conf.d/wifi-powersave.conf
```

Insert:

```ini id="nj0b0u"
[connection]
wifi.powersave = 2
```

Restart network manager:

```bash id="7hrc92"
sudo systemctl restart NetworkManager
```

---

# 2. Auto-reboot after power outage

Enable automatic boot when power returns.

## BIOS setting

Enter BIOS:

1. reboot T450
2. press:

```text id="q6ru3s"
Enter
```

3. then press:

```text id="scjy68"
F1
```

Navigate:

```text id="pvw6be"
Config
→ Power
→ After Power Loss
```

Set:

```text id="j42y7v"
Power On
```

Save:

```text id="6ib5bh"
F10
```

---

# 3. BIOS settings for always-on server behaviour

Recommended BIOS configuration:

### disable sleep triggers

```text id="k37hkr"
Config
→ Power
→ Intel SpeedStep → Enabled
→ CPU Power Management → Enabled
```

### disable deep sleep instability

```text id="skk0iy"
Config
→ Power
→ Sleep State → Linux
```

### allow lid-closed operation

```text id="eq04mp"
Config
→ Display
→ Boot Display Device → ThinkPad LCD
```

### prevent USB power drop

```text id="w9n6py"
Config
→ USB
→ Always On USB → Enabled
```

---

# 4. CPU power optimization (lower electricity usage)

Install power tuning tools:

```bash id="0q4u0j"
sudo apt install -y powertop cpufrequtils
```

---

## enable ondemand governor

```bash id="nvq0n8"
sudo nano /etc/default/cpufrequtils
```

Insert:

```ini id="t3l5vs"
GOVERNOR="ondemand"
```

Apply:

```bash id="8jz3q9"
sudo systemctl enable cpufrequtils
sudo systemctl start cpufrequtils
```

---

## auto-tune power settings

Run once:

```bash id="xtjl6t"
sudo powertop --auto-tune
```

To apply automatically at boot:

```bash id="1vlxtv"
sudo nano /etc/systemd/system/powertop.service
```

Insert:

```ini id="3k9p7k"
[Unit]
Description=Powertop tunings

[Service]
Type=oneshot
ExecStart=/usr/sbin/powertop --auto-tune

[Install]
WantedBy=multi-user.target
```

Enable:

```bash id="33zqwc"
sudo systemctl enable powertop
```

---

# 5. Optional: disable unused hardware

Check devices:

```bash id="3hp6d5"
lspci
```

Typical candidates to disable:

* bluetooth
* camera
* audio
* fingerprint reader

Example disable bluetooth:

```bash id="n0xv3v"
sudo systemctl disable bluetooth
```

---

# 6. Verify stable headless behaviour

Test checklist:

### close lid test

```text id="f4t1vr"
close lid → wait 2 minutes → SSH still connected
```

### reboot persistence

```bash id="wycqlx"
sudo reboot
```

After reboot:

```bash id="bc1g6x"
ssh ngminhtrung@192.168.1.16
```

### docker auto-start

```bash id="m8b0wf"
docker ps
```

---

# 7. Expected power consumption

Typical ThinkPad T450 headless:

| state                 | watts  |
| --------------------- | ------ |
| idle                  | 6–8W   |
| light docker workload | 8–12W  |
| moderate load         | 12–18W |

Much lower than desktop server.

---

# Final recommended baseline

```text id="ej1yx7"
Debian headless
Docker
Tailscale
n8n
powertop tuned
lid closed 24/7
```

---

If useful, next step can include:

* static IP configuration
* automated backup of docker volumes
* monitoring stack (Netdata / Prometheus)
* remote wake-up (Wake-on-LAN)
* resource limits for docker containers
