# GUFW TOTAL CONTROLLER

### *It controls everything.*

![Linux](https://img.shields.io/badge/platform-Linux-blue?style=for-the-badge&logo=linux)
![Rust](https://img.shields.io/badge/built_with-Rust-orange?style=for-the-badge&logo=rust)
![License](https://img.shields.io/badge/license-GPL--3.0-green?style=for-the-badge)
![AppImage](https://img.shields.io/badge/distro-AppImage-red?style=for-the-badge)

> One button. Total firewall lockdown. A dark neon kill-switch controller that
> hardens UFW, guards every packet, and launches Riseup VPN — automatically,
> on every Linux distribution.

---

## Table of Contents

- [What This Is](#what-this-is)
- [Features](#features)
- [How It Works](#how-it-works)
- [The Kill-Switch Ruleset](#the-kill-switch-ruleset)
- [Smart Rule Management](#smart-rule-management)
- [Supported Distributions & Package Managers](#supported-distributions--package-managers)
- [Quick Start](#quick-start)
- [Usage Guide](#usage-guide)
- [Building From Source](#building-from-source)
- [Security Model](#security-model)
- [Project Structure](#project-structure)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)

---

## What This Is

**GUFW TOTAL CONTROLLER** is a native Linux desktop application, written in
Rust, that turns UFW (Uncomplicated Firewall) into a fully automatic VPN
kill-switch — with a single button as the entire interface.

The threat model is simple: if your VPN drops, not one packet may leave your
machine over the open internet. This app enforces that by default-denying
**all** incoming, outgoing, and forwarded traffic, then opening narrow,
explicit exceptions only for localhost, the VPN tunnel interfaces (`tun+`,
`wg+`), essential VPN handshake ports, and your virtual-machine bridge.
When you need traffic, one press opens outgoing and auto-launches Riseup VPN;
one OK click slams the kill-switch shut again.

Everything is automatic: distro detection, GUFW/UFW installation, rule
deployment, Riseup VPN installation, and VPN launch. You never touch a
terminal.

---

## Features

- **Fully automatic setup** — detects your distro, installs GUFW/UFW with the
  native package manager, deploys the kill-switch rules, and installs Riseup
  VPN, all on first launch with nothing but privilege prompts.
- **One-button operation** — the entire UI is a single button:
  `ALLOW OUTGOING + LAUNCH RISEUP VPN`. Press it, traffic opens and Riseup
  VPN starts by itself.
- **OK-only safety box** — after outgoing opens, a message box appears with a
  single OK button. Clicking OK immediately re-denies outgoing and restores
  full kill-switch protection. You cannot forget to close it.
- **Smart rule management** — every launch bash-checks (`ufw status verbose`)
  whether the kill-switch rules are already active. Present → skipped, never
  re-added. Missing → applied. First-time installs get rules immediately.
- **Zero-backup policy** — no firewall backup is ever taken before applying
  rules, and stale UFW backup files (`*.bak`, `*.backup`, `*.orig`, `*~`)
  are automatically cleaned up. The live rules are never touched by cleanup.
- **Riseup VPN integration** — detects `riseup-vpn` (plus `bitmask`, snap,
  and flatpak variants); installs it from the official APT repository or PPA
  on Debian-family systems, AUR helpers on Arch, native managers elsewhere,
  with snap fallback everywhere.
- **Borderless neon UI** — custom title bar with minimize / maximize-restore /
  close controls on the top-left, glowing red title, glowing green subtitle,
  live activity log. At home on KDE Plasma (X11 and Wayland) and every other
  desktop.
- **Portable AppImage** — one file, no installation, runs on virtually any
  modern 64-bit distribution.

---

## How It Works

```
 ┌───────────┐   ┌──────────────┐   ┌───────────────┐   ┌──────────────┐
 │ AUTO-SETUP │──▶│ RULE CHECK   │──▶│ SINGLE BUTTON │──▶│  OK = LOCK  │
 │ distro→    │   │ rules there? │   │ allow outgoing│   │ deny out-   │
 │ gufw→rules │   │ yes: skip    │   │ + auto-launch │   │ going again │
 │ →riseup    │   │ no: add      │   │ Riseup VPN    │   │             │
 └───────────┘   └──────────────┘   └───────────────┘   └──────────────┘
```

1. **Launch the app.** It detects your Linux (`/etc/os-release`), picks the
   package manager, installs GUFW/UFW if missing, and deploys the kill-switch
   rules — but only if a bash check proves they are not already active.
2. **Press the button.** Outgoing is allowed (`ufw default allow outgoing` +
   reload) and Riseup VPN launches automatically, detached — no terminal.
3. **Read the message box.** It confirms outgoing is open and VPN is up, with
   one OK button.
4. **Click OK.** Outgoing is denied again (`ufw default deny outgoing` +
   reload). Kill-switch restored. Done.

---

## The Kill-Switch Ruleset

The exact rules deployed (no more, no less):

```bash
ufw --force reset
ufw default deny incoming
ufw default deny outgoing
ufw default deny FORWARD        # + routed fallback for modern UFW
ufw allow in on lo to any
ufw allow out on lo to any
ufw allow out on tun+ to any    # VPN tunnels fully trusted
ufw allow in on tun+ to any
ufw allow out on wg+ to any
ufw allow in on wg+ to any
ufw allow out 53/udp           # DNS for VPN handshake only
ufw allow out 53/tcp
ufw allow out 1194/udp         # OpenVPN (Riseup)
ufw allow out 1194/tcp
ufw allow out 51820/udp        # WireGuard
ufw allow in on virbr0          # Virt-Manager bridge
ufw allow out on virbr0
ufw route allow in on virbr0 out on tun+
ufw route allow in on tun+ out on virbr0
ufw --force enable
ufw reload
```

Design notes:

- **Default-deny everything** is the foundation — incoming, outgoing, and
  forwarding are all closed unless explicitly punched above.
- **Only the tunnel carries traffic.** Ports 80/443/Tor/DNS ride inside
  `tun+`/`wg+`; the physical interface exposes nothing but the VPN handshake
  (DNS + 1194 + 51820).
- **Loopback stays open** — strictly for internal system needs.
- **VMs route through the tunnel** — `virbr0` traffic may only exit via
  `tun+`, so guests inherit the kill-switch.

---

## Smart Rule Management

Re-adding rules on every launch would be slow, noisy, and risky. Instead:

| Situation | Behavior |
|-----------|----------|
| Rules already active (firewall up, deny in/out, `tun+` + `51820` present) | **Skipped.** Log shows “already present — skipping”. |
| Fresh GUFW install, or rules missing/changed | Stale UFW backups deleted, full ruleset applied, firewall enabled. |
| Before every apply | **No backup is taken, ever.** Cleanup removes `*.bak`, `*.backup`, `*.orig`, `*~` under `/etc/ufw` and `/var/lib/ufw` — live rules untouched. |

---

## Supported Distributions & Package Managers

| Family | Manager | GUFW install | Riseup install |
|--------|---------|--------------|----------------|
| Debian, Parrot, Kali, Mint (Debian ed.), LMDE | APT | `apt-get install -y ufw gufw` | `apt-get install -y riseup-vpn` (native repo) |
| Ubuntu, Neon, Pop!_OS, Zorin, Mint (Ubuntu ed.), elementary | APT | same as above | LEAP PPA `ppa:leapcodes/riseup-vpn`, then `apt install riseup-vpn` |
| Fedora, RHEL 8+, Rocky, AlmaLinux | DNF | `dnf install -y ufw gufw` | `dnf install -y riseup-vpn` → snap fallback |
| RHEL 7, CentOS 7 | YUM | `yum install -y ufw gufw` | `yum install -y riseup-vpn` → snap fallback |
| Arch, Manjaro, EndeavourOS, Garuda, CachyOS | Pacman | `pacman -Sy ufw gufw` | AUR via `yay`/`paru` → snap fallback |
| openSUSE | Zypper | `zypper install ufw gufw` | `zypper install riseup-vpn` → snap fallback |
| Alpine, Void, Solus, Gentoo | APK / XBPS / EOPKG / Emerge | native install | snap fallback |
| Anything with snapd | — | — | `snap install riseup-vpn --classic` |

Privilege escalation uses `pkexec` first (graphical Plasma/GNOME prompt),
then `sudo`, then direct execution when already root.

---

## Quick Start

### Option A — AppImage (recommended)

1. Download `GUFW_TOTAL_CONTROLLER-x86_64.AppImage` from the
   [Releases](../../releases) page.
2. Make it executable:
   ```bash
   chmod +x GUFW_TOTAL_CONTROLLER-x86_64.AppImage
   ```
3. Run it:
   ```bash
   ./GUFW_TOTAL_CONTROLLER-x86_64.AppImage
   ```
4. Approve the privilege prompts during automatic setup, then press the
   single button when you need traffic.

### Requirements

- 64-bit Linux (x86_64), X11 or Wayland.
- `polkit` (`pkexec`) or `sudo` for firewall and install operations.
- Internet access on first run (package installation).

---

## Usage Guide

| Control | Action |
|---------|--------|
| **ALLOW OUTGOING + LAUNCH RISEUP VPN** | The only button. Opens outgoing, reloads UFW, auto-launches Riseup VPN, shows the message box. Disabled while setup or a job runs. |
| **OK** (in the OUTGOING STATUS box) | The only other button. Closes outgoing, reloads UFW, restores the kill-switch. |
| Status panel | Linux, package manager, GUFW, firewall, Riseup VPN, and outgoing state at a glance. |
| Activity log | Read-only terminal-style log of every check, install, rule change, and launch. |
| `−` `□` `✕` (top-left) | Minimize, maximize/restore toggle, close — custom borderless title bar. |

---

## Building From Source

```bash
# 1. Install Rust (https://rustup.rs) plus GUI build deps, e.g. on Debian/Ubuntu:
#    sudo apt install build-essential pkg-config libgl1-mesa-dev libxkbcommon-dev

# 2. Clone and build
git clone <your-repo-url>
cd gufw-total-controller
cargo build --release

# 3. Package the AppImage (fetches appimagetool on first run)
./build-appimage.sh
```

Output:

- `GUFW_TOTAL_CONTROLLER-x86_64.AppImage` — the portable application.
- `GUFW_TOTAL_CONTROLLER-portable-linux.tar.gz` — fallback bundle, created
  automatically only if AppImage packaging is unavailable.

---

## Security Model

- The UI **never runs as root**. Privilege is escalated per-operation via
  `pkexec`/`sudo`, only for installs, rule changes, and status reads.
- **Default-deny is the resting state.** Outgoing is open only in the short
  window between the button press and the OK click — a window the message
  box makes impossible to forget.
- No backups means no stale rulesets lying around to be accidentally
  restored; the live configuration is the single source of truth.
- Dependency/rule names are never interpolated from untrusted input; all
  privileged scripts are fixed strings staged through temp files.
- Every privileged command streams into the visible activity log.

---

## Project Structure

```
gufw-total-controller/
├── src/main.rs                        # Entire application (GUI + backend)
├── Cargo.toml / Cargo.lock            # Rust manifest (eframe/egui)
├── assets/
│   ├── icon.svg / icon.png            # Neon shield icon
│   └── gufw-total-controller.desktop  # Desktop entry
├── build-appimage.sh                  # Release build + AppImage packaging
├── README.md                          # This file
├── RELEASE_NOTES.md                   # Release notes (per version)
└── target/release/                    # Build output (git-ignored)
```

> Historical note: this folder previously hosted an unrelated
> Auto Package Installer experiment; its code is preserved intact in
> `../auto-package-installer-backup/` and shares nothing with this app.

---

## Roadmap

- UFW status auto-refresh indicator in the status panel.
- Optional WireGuard-only strict mode (drop `tun+`/OpenVPN paths).
- Per-rule toggle view (read-only inspector, still one-button philosophy).
- Flatpak distribution alongside AppImage.
- English-only UI is intentional and stays.

---

## Contributing

Issues and pull requests are welcome. Please:

1. Run `cargo build` (and `cargo fmt` if you touch formatting) before
   submitting.
2. Keep the one-button philosophy: no new buttons without a strong reason.
3. Never take firewall backups — cleanup, don't archive.
4. Test rule changes on a VM first; a bad default-deny locks you out.

---

## License

GPL-3.0-or-later. If you distribute a modified AppImage, keep the
release-notes habit alive — future you will say thanks.
