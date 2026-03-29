# Homelab Setup Notes

This document reorganizes the original chat transcript into a cleaner topic flow. Each section captures a distinct question and the relevant answer.

Related note:

- [Debian Minimization for Homelab](/Users/travisnguyen/Documents/Coursera/homelab/debian-minimization.md)

## Recommended Topic Sequence

1. Create a bootable USB from macOS
2. Choose the best distro for the ThinkPad T450 homelab server
3. Install a minimal headless Debian system
4. Define the target homelab architecture
5. Decide how to remotely manage the server from VS Code Codex

---

## Topic 1: How do I create a bootable Linux USB from macOS?

### Short answer

Use native macOS tools: `diskutil` and `dd`.

### Recommended use case

This is useful when you need installer media for Debian, Ubuntu Server, Linux Mint, or Proxmox.

### Steps

#### 1. Download the ISO

Examples:

- Debian: `debian-12.x.x-amd64-netinst.iso`
- Ubuntu Server: `ubuntu-24.04-live-server-amd64.iso`
- Linux Mint Cinnamon: `linuxmint-21.3-cinnamon-64bit.iso`

Optional checksum verification:

```bash
shasum -a 256 ~/Downloads/linuxmint-21.3-cinnamon-64bit.iso
```

#### 2. Identify the USB device

```bash
diskutil list
```

Example result:

```text
/dev/disk4 (external, physical)
```

#### 3. Unmount the USB

```bash
diskutil unmountDisk /dev/disk4
```

#### 4. Write the ISO

Use the raw device (`rdisk`) for better speed:

```bash
sudo dd if=~/Downloads/linuxmint-21.3-cinnamon-64bit.iso of=/dev/rdisk4 bs=4m status=progress
```

#### 5. Eject the USB

```bash
diskutil eject /dev/disk4
```

### Minimal command summary

```bash
diskutil list
diskutil unmountDisk /dev/diskX
sudo dd if=linux.iso of=/dev/rdiskX bs=4m status=progress
diskutil eject /dev/diskX
```

### Common issues

| Issue | Likely cause | Fix |
| --- | --- | --- |
| USB does not boot | Wrong device written | Re-run `diskutil list` and confirm disk number |
| `dd` is very slow | Used `/dev/diskX` instead of `/dev/rdiskX` | Use the raw device |
| Mac refuses to boot external media | Secure Boot or external boot policy | Allow external boot in recovery |
| Installer fails unexpectedly | Corrupted ISO | Verify SHA256 checksum |

---

## Topic 2: What distro should I use for a ThinkPad T450 headless homelab server?

### Short answer

Use **Debian minimal**. For the installer USB, use the **Debian netinst ISO**.

### Why Debian is the best fit

- Minimal resource usage
- Stable and predictable for always-on workloads
- Strong Docker and homelab ecosystem support
- Good fit for older hardware like the ThinkPad T450
- Closer to a clean server baseline than desktop distros

### Recommended ranking

#### 1. Debian minimal

Best default choice for a headless homelab server.

Why:

- Lowest overhead
- High stability
- Strong learning value
- Excellent flexibility for Docker-based services

Recommended installer:

```text
debian-12.x.x-amd64-netinst.iso
```

#### 2. Ubuntu Server LTS

Good alternative if you want an easier installer and faster initial setup.

Tradeoff:

- Slightly heavier than Debian
- May include defaults you do not want in a minimal homelab

#### 3. Proxmox VE

Choose this only if your main goal is virtualization and running multiple VMs or LXC containers.

Tradeoff:

- More overhead
- Better suited to VM-heavy usage than a simple Docker node

#### 4. DietPi

Useful if you want an ultra-light setup with prepackaged home-server options.

Tradeoff:

- Less of a pure Debian server experience

### Distros not recommended for this use case

| Distro | Why not |
| --- | --- |
| Linux Mint | Desktop-oriented and unnecessary for a headless server |
| Ubuntu Desktop | Large ISO and unnecessary GUI overhead |
| Fedora Workstation | Shorter lifecycle for this use case |
| Arch | Too manual for a straightforward stable homelab baseline |
| Kali | Not intended for general server operation |

### Decision matrix

| Criteria | Debian | Ubuntu Server | Proxmox | DietPi |
| --- | --- | --- | --- | --- |
| Minimal footprint | Best | Good | Medium | Best |
| Stability | Best | Very good | Very good | Good |
| Docker homelab fit | Best | Best | Medium | Good |
| Learning value | Best | Good | Medium | Medium |
| Maintenance effort | Medium | Low | Medium | Low |

---

## Topic 3: What is the best installer USB if the final goal is Debian on the T450?

### Short answer

Use the **Debian netinst ISO** directly. No intermediate distro is needed.

### Reasoning

The bootable USB only needs to install Debian onto the ThinkPad T450. The ideal installer therefore should be:

- Stable
- Small
- Compatible with older Intel hardware
- Easy to use for a minimal headless setup
- Reliable for GRUB and partitioning

### Final recommendation

```text
debian-12.x.x-amd64-netinst.iso
```

Download:

- `https://www.debian.org/download`

---

## Topic 4: How should I install Debian for headless homelab use?

### Short answer

Install Debian without a desktop environment. Enable SSH during setup and keep the base system minimal.

For a dedicated low-RAM and low-power tuning checklist, see [Debian Minimization for Homelab](/Users/travisnguyen/Documents/Coursera/homelab/debian-minimization.md).

### Boot workflow

#### 1. Create the Debian installer USB on macOS

```bash
diskutil list
diskutil unmountDisk /dev/diskX
sudo dd if=debian-12.x.x-amd64-netinst.iso of=/dev/rdiskX bs=4m status=progress
diskutil eject /dev/diskX
```

#### 2. Boot the ThinkPad T450

1. Insert the USB
2. Power on
3. Press `F12`
4. Select the USB boot device

#### 3. Choose installer options for a headless server

Software selection:

```text
[ ] Debian desktop environment
[x] SSH server
[x] standard system utilities
```

Disk layout:

```text
/
swap
```

Recommended filesystem:

```text
ext4
```

### Post-install baseline packages

```bash
sudo apt update
sudo apt install \
  curl \
  git \
  htop \
  ca-certificates \
  gnupg \
  docker.io \
  docker-compose \
  fail2ban
```

Ensure SSH is enabled:

```bash
sudo systemctl enable ssh
sudo systemctl start ssh
```

### Resulting baseline

- Debian 12 minimal
- SSH access enabled
- Docker ready
- Good base for automation and self-hosted services

---

## Topic 5: What should the target homelab architecture look like?

### Short answer

Use the ThinkPad T450 as a lightweight Debian Docker host.

### Recommended target state

```text
ThinkPad T450
  Debian 12 minimal
    Docker
      n8n
      AdGuard
      reverse proxy
      Postgres
      monitoring stack
```

### Why this architecture works

- Fits the hardware limits of the T450
- Keeps the system simple to operate
- Supports incremental growth through containers
- Works well with headless remote management

### Suggested remote access baseline

- SSH for direct management
- Tailscale for secure remote connectivity

---

## Topic 6: Can VS Code Codex manage the headless Debian server over SSH?

### Short answer

Yes. The most relevant path discussed in the original notes is an MCP-based SSH integration.

### Recommended option

#### MCP SSH Manager

Typical capabilities:

- Execute remote shell commands
- Read logs
- Upload and download files
- Inspect metrics
- Run Docker commands
- Manage multiple servers

### Example actions from Codex

```text
check disk usage on t450
restart docker container n8n
show last 200 lines of syslog
update packages
git pull repo
```

### Conceptual architecture

```text
VS Code + Codex
  MCP SSH server
    SSH connection
      ThinkPad T450 Debian
        Docker containers
```

### Example MCP registration

File:

```text
.vscode/mcp.json
```

Example:

```json
{
  "mcpServers": {
    "ssh": {
      "command": "mcp-ssh-manager"
    }
  }
}
```

### Complementary tools

| Function | Tool |
| --- | --- |
| Remote terminal | VS Code Remote SSH |
| Secure remote network access | Tailscale |
| Container UI | Portainer |
| Logs | Dozzle |
| Monitoring | Netdata |

### Recommendation

For this homelab, the cleanest remote management path is:

- Debian minimal on the T450
- SSH as the core access method
- Tailscale for connectivity
- MCP SSH integration if you want Codex-driven server operations

---

## Final Consolidated Recommendation

If the goal is to turn the ThinkPad T450 into a headless homelab server:

1. Create a bootable USB from macOS using `diskutil` and `dd`
2. Use the Debian netinst ISO as the installer
3. Install Debian minimal with `SSH server` and `standard system utilities` only
4. Add Docker and a small baseline package set after install
5. Run homelab services as containers
6. Manage the server through SSH, optionally extended with MCP in VS Code Codex

This is the cleanest sequence from setup to operation.
