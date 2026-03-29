n# Debian Minimization for Homelab

This note pulls out the optimization topic that is only implied in [set-up homelab.md](/Users/travisnguyen/Documents/Coursera/homelab/set-up%20homelab.md). The original file recommends Debian minimal and a headless install, but it does not spell out how to reduce RAM use or idle power further.

## Scope

This guide focuses on a Debian-based ThinkPad T450 homelab node that should:

- use as little RAM as practical
- avoid unnecessary background activity
- reduce idle power draw where possible
- still remain simple and stable for Docker-hosted services

---

## Topic 1: What is the biggest lever for reducing RAM and power use?

### Short answer

Install less software.

### Most important decisions

- Do not install a desktop environment
- Do not install packages you will not use
- Run only the services you actually need
- Prefer one simple Docker host over a VM-heavy stack

### Why this matters

The biggest savings usually come from avoiding GUI components, extra daemons, and unnecessary containers rather than from micro-tuning the kernel.

---

## Topic 2: How should Debian be installed for the lowest practical footprint?

### Short answer

Use the Debian netinst ISO and select only the minimum task set needed for a headless machine.

### Recommended installer choices

Software selection:

```text
[ ] Debian desktop environment
[x] SSH server
[x] standard system utilities
```

### Recommended partitioning

Simple layout:

```text
/
swap
```

### Why this helps

- No GUI packages
- Fewer services started at boot
- Lower memory usage
- Less disk activity

Reference back: [set-up homelab.md](/Users/travisnguyen/Documents/Coursera/homelab/set-up%20homelab.md)

---

## Topic 3: What should I avoid installing if I want lower RAM use?

### Short answer

Avoid anything that adds a user session, indexing, auto-discovery, or a web UI you do not need.

### Usually unnecessary on a headless homelab

- desktop environments
- display managers
- Bluetooth tools if Bluetooth is not used
- printer services
- audio stacks
- graphical package managers
- multiple monitoring dashboards at once

### Practical rule

If a package is only useful from a local screen on the laptop, it probably does not belong on a headless server.

---

## Topic 4: Which services should I keep disabled unless I need them?

### Short answer

Only keep core remote-management and container services enabled by default.

### Usually keep enabled

- `ssh`
- `docker`
- `systemd-networkd` or your chosen network service

### Review carefully before enabling

- `bluetooth`
- `cups`
- `avahi-daemon`
- `ModemManager`
- `wpa_supplicant` if you use only wired Ethernet

### How to inspect enabled services

```bash
systemctl list-unit-files --state=enabled
```

### How to disable a service

```bash
sudo systemctl disable --now SERVICE_NAME
```

### Caution

Do not disable networking, storage, or SSH services unless you are certain about the impact.

---

## Topic 5: How can Docker usage be kept lightweight?

### Short answer

Run fewer containers, use smaller stacks, and avoid infrastructure you do not need yet.

### Good practice

- Start with only the services you actively use
- Avoid running multiple admin dashboards that overlap
- Add Postgres only when a service actually benefits from it
- Avoid running full virtualization if Docker is enough

### Example lightweight stack

```text
Debian
  Docker
    n8n
    reverse proxy
    one monitoring/logging tool
```

### Why this matters

Each container adds memory overhead, startup activity, logs, and maintenance work.

---

## Topic 6: What helps reduce energy use on an always-on laptop server?

### Short answer

Reduce unnecessary work, prefer wired networking when practical, and let the CPU stay idle as much as possible.

### Practical steps

- keep the machine headless
- use Ethernet instead of Wi-Fi if available
- disable radios you do not need, such as Bluetooth
- avoid constant heavy monitoring intervals
- avoid unnecessary containers that poll or scan frequently
- keep CPU-intensive tasks off the T450 unless needed

### ThinkPad-specific mindset

The T450 saves power best when it is doing very little most of the time. A simple Docker host idles more efficiently than a laptop running a desktop session or many background daemons.

---

## Topic 7: Are there safe software tweaks worth doing after install?

### Short answer

Yes, but keep them conservative.

### Reasonable post-install steps

Update the system:

```bash
sudo apt update
sudo apt upgrade
```

Install only a small baseline:

```bash
sudo apt install \
  curl \
  git \
  htop \
  ca-certificates \
  docker.io \
  docker-compose
```

Review memory usage:

```bash
free -h
ps aux --sort=-%mem | head
```

Review active services:

```bash
systemctl --type=service --state=running
```

### Optional swap note

If RAM is limited, keep some swap configured for stability. Zero-swap is not automatically better for a small homelab node.

---

## Topic 8: What is the practical minimum configuration for this homelab?

### Short answer

For the T450, the clean minimum is:

- Debian minimal
- no desktop
- SSH enabled
- Docker enabled
- only a few containers
- no extra local GUI tools

### Recommended starting point

```text
ThinkPad T450
  Debian minimal
    ssh
    docker
      n8n
      reverse proxy
```

Then add services only when there is a clear need.

---

## Final Recommendation

The original setup guide already points in the right direction by recommending Debian minimal and a headless install. The missing detail is that minimization should be treated as an operating principle:

1. Install the fewest packages possible
2. Enable the fewest services possible
3. Run the fewest containers possible
4. Keep the machine idle most of the time

That is the most reliable way to reduce both RAM usage and energy consumption on a Debian homelab node.
