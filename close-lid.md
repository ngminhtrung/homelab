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
