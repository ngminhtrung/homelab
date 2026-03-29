# Debian Minimization for Homelab

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

---

## Topic 9: How can RAM and energy use be monitored over time with graphs?

### Short answer

Use one lightweight metrics stack for graphing and long-term storage.

### Recommended stack

For this homelab, the most practical option is:

- `Prometheus` for time-series storage
- `Grafana` for dashboards and graphs
- `Node Exporter` for host RAM and CPU metrics
- one small custom collector or script for power-related metrics

### Why this is the best fit

- Prometheus stores metrics over time
- Grafana gives graphs by hour, day, week, or month
- Node Exporter is standard and lightweight
- You can keep all metrics on the same Debian host or move them later

### What can be measured reliably

Reliable and easy:

- RAM used
- RAM available
- swap usage
- CPU utilization
- load average
- disk I/O
- network traffic

More hardware-dependent:

- battery discharge rate
- AC power state
- estimated power draw

On a ThinkPad T450, power measurement quality depends on what Linux exposes through battery and ACPI interfaces.

---

## Topic 10: What is the simplest graphing solution if I do not want to build much?

### Short answer

Use `Netdata` if you want fast setup with built-in graphs.

### Why choose Netdata

- very quick to install
- built-in dashboards
- host RAM graphs immediately
- easy to inspect trends

### Tradeoff

- easier to start, but less flexible than Prometheus + Grafana for custom reporting
- long-term retention and custom power metrics usually need more tuning

### Recommendation

- Choose `Netdata` if you want the fastest path to visual monitoring
- Choose `Prometheus + Grafana` if you want a cleaner long-term metrics platform

---

## Topic 11: What is the recommended monitoring design for this homelab?

### Short answer

Use Prometheus + Grafana for the main history and dashboards, and keep power collection separate from normal system metrics.

### Suggested architecture

```text
ThinkPad T450
  Debian
    node_exporter
    power-metrics script or collector
    prometheus
    grafana
```

### Logged data flow

```text
host metrics
  -> exporters / scripts
  -> Prometheus
  -> Grafana dashboards
```

### Best split of responsibility

- `Node Exporter`: RAM, CPU, disk, filesystem, load
- custom power metric source: battery rate, AC state, estimated draw
- `Prometheus`: stores history
- `Grafana`: graphs and reporting

---

## Topic 12: How do I graph RAM consumption over time?

### Short answer

Use `Node Exporter` and graph memory metrics in Grafana.

### Typical useful RAM metrics

- total memory
- available memory
- used memory
- cached memory
- swap used

### Practical Grafana panels

- RAM used over 24 hours
- RAM available over 7 days
- swap use over 30 days
- top memory spikes correlated with container restarts or backup jobs

### Why this matters

For minimization work, the key question is not just current RAM use. It is whether RAM usage drifts upward over time or spikes during backups, updates, or workflow runs.

---

## Topic 13: How do I monitor energy or power usage on a ThinkPad T450?

### Short answer

Use the best metric source available on the hardware, but treat power data as more approximate than RAM data.

### Practical options

#### Option 1: Battery and ACPI-based reporting

If the laptop still has a working battery, Linux can often expose:

- battery charge level
- charging or discharging state
- discharge rate

This can be sampled over time and stored as metrics.

#### Option 2: `powerstat`

`powerstat` can estimate power-related behavior over time and is useful for periodic logging.

#### Option 3: CPU package power interfaces

Some systems expose power information through kernel interfaces, but availability varies and should not be assumed.

### Recommendation

For a T450, the most realistic approach is:

- use RAM and CPU metrics as primary optimization signals
- add battery or ACPI-based power metrics if available
- use `powerstat` samples for trend analysis rather than absolute precision

---

## Topic 14: Where should the monitoring data be logged?

### Short answer

Store it in Prometheus and visualize it in Grafana.

### Good retention targets

- 24 hours for detailed troubleshooting
- 7 to 30 days for optimization trends
- longer if disk space allows

### What to keep long-term

- RAM usage trends
- swap trends
- CPU idle versus busy periods
- estimated power trend by day
- service or container count over time

### Why this is useful

This gives you evidence for decisions such as:

- whether a new container increased idle RAM
- whether disabling Wi-Fi reduced average power use
- whether monitoring itself is adding too much overhead

---

## Topic 15: What is a practical low-overhead setup to start with?

### Short answer

Start with one of these two paths.

### Option A: Fastest setup

- `Netdata`
- use its built-in host graphs
- use it first to see whether deeper monitoring is even needed

### Option B: Better long-term setup

- `Prometheus`
- `Grafana`
- `Node Exporter`
- one scheduled script for power metrics

### Suggested decision

For your homelab, the better long-term path is:

```text
Prometheus + Grafana + Node Exporter + small power metric script
```

This keeps the stack flexible without becoming unnecessarily large.

---

## Topic 16: What should a simple logging approach look like for power metrics?

### Short answer

Sample power-related data on an interval and expose it as metrics or append it to a small log that Prometheus can scrape indirectly.

### Example approach

1. Run a script every 1 to 5 minutes
2. Read available values from battery or ACPI interfaces
3. Record values such as:
   `timestamp`, `battery_status`, `battery_power_w`, `ac_online`
4. Export those values for Prometheus collection

### Practical note

If battery power reporting is unavailable or unreliable, you can still compare changes using:

- CPU idle percentage
- package temperature
- average load
- whether radios and unnecessary services are disabled

Those are indirect signals, but they are often enough to validate minimization work.
4. Keep the machine idle most of the time

That is the most reliable way to reduce both RAM usage and energy consumption on a Debian homelab node.
