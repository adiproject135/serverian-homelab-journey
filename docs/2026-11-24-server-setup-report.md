# Debian Server Setup Report

**Work period:** 23-24 November 2026  
**Host:** `adrian-server`  
**Distribution:** Debian GNU/Linux 13 (trixie)  
**Primary account:** `serverian`  
**Purpose:** Document the server configuration work completed today, the reasoning behind the decisions, the problems encountered, and the next steps.

---

## 1. Executive summary

The work across 23-24 November turned a fresh Debian 13 installation into a remotely accessible server. The first major task alone consumed approximately three hours: getting the newly installed system online, diagnosing why network access was only partially working, restoring DNS, and moving from temporary USB tethering to the intended Wi-Fi connection. The objectives were:

- Make remote administration reliable even when the local network changes.
- Secure SSH access and avoid exposing unnecessary services.
- Introduce a simple, restrictive firewall.
- Keep the machine available when nobody is interacting with it.
- Reuse my existing KDE configuration to reduce setup time and preserve a familiar desktop environment.

The result is a Debian 13 machine with working Wi-Fi and DNS, SSH key authentication, Tailscale connectivity, an active nftables firewall, a restricted administrative helper, migrated KDE preferences, and a persistent sleep inhibitor.



## 2. Installation networking: the three-hour connectivity problem

**This was the first major obstacle and a substantial part of the work.** The Debian installation was not immediately usable as a networked machine. Roughly three hours were spent diagnosing connectivity, first at installation time and then from the installed system.

### 2.1 Starting point: Debian had no usable package sources

The installed system was Debian 13 (Trixie), release 13.8, running kernel `6.12.117`. The APT source configuration was effectively empty:

- `/etc/apt/sources.list` contained only comments.
- The installer-media `deb cdrom` entries were disabled.
- There were no active network repositories available to APT.

This meant that even after obtaining a network connection, package installation and updates still depended on configuring valid repository sources.

### 2.2 Wi-Fi was detected, but not connected

The wireless interface was present as `wlp1s0`, but it was DOWN. I tried bringing it up:

```bash
sudo ip link set wlp1s0 up
```

The `iw` utility was not installed, so the expected wireless inspection command was unavailable:

```bash
iw dev
```

This was an early complication: the hardware interface existed, but the minimal installation did not have all the convenient diagnostic tools, and the machine was not yet connected to Wi-Fi.

### 2.3 Temporary workaround: USB tethering

I switched to USB tethering through a phone to obtain a temporary network connection and continue troubleshooting.

A USB network interface appeared. The interface name changed during the troubleshooting records: an early interface was recorded as the USB Ethernet interface; the later working tethering interface was the USB Ethernet interface.

The interface initially appeared DOWN. It was brought up with:

```bash
sudo ip link set <usb-interface> up
```

The interface then reported state `UNKNOWN`. DHCP and routing troubleshooting followed. The working tethering connection eventually received:

- Interface: the USB Ethernet interface
- IPv4 address: the USB-tethered IPv4 address

This established that the phone-to-Debian USB network path could provide an address.

### 2.4 The misleading part: IP connectivity worked, DNS did not

The key diagnostic distinction was between reaching an IP address and resolving a domain name.

A direct connectivity test succeeded:

```bash
ping 1.1.1.1
```

But DNS resolution failed. The underlying issue identified in the session was that `/etc/resolv.conf` was absent. In other words, the machine had an IP address and could reach an external IP, but it had no resolver configuration telling it which DNS servers to ask.

I created `/etc/resolv.conf` with public DNS resolvers:

```conf
nameserver 1.1.1.1
nameserver 8.8.8.8
```

After that, APT could resolve repository hostnames and download packages. An `apt update` run successfully downloaded approximately 17.2 MB of package metadata.

This was the decisive fix for the apparent "internet does not work" problem: the network path itself was functioning, while name resolution was not.

### 2.5 Repository and firmware warnings

Even after APT started working, the package/repository setup was not entirely clean. Firmware-related warnings remained, and a later warning exposed a misspelled repository component:

```
non-free-firware0
```

The component name was misspelled. This matters because APT repository components must match the repository's actual component names; a typo can prevent the intended package index from being fetched.

The firmware warnings were not the same problem as the missing DNS configuration. They were separate cleanup items and should not be conflated with the connectivity fix.

### 2.6 Why SSH from Fedora initially failed

USB tethering placed Debian and Fedora on different IP subnets:

| Device | Interface / connection | Address |
|---|---|---|
| Fedora workstation | Wi-Fi | an address on the local Wi-Fi subnet |
| Debian server | USB tethering | `USB-tethered subnet address` |

The devices were not on the same local subnet. A direct SSH attempt from Fedora to the Debian tethering address therefore did not provide the intended straightforward local connection path. A ping from Fedora to the USB-tethered IPv4 address was also attempted during diagnosis.

This was not an SSH key problem. The two machines' network topology was the issue.

### 2.7 Moving from temporary tethering to Wi-Fi

Once package access was available, I installed/enabled NetworkManager and connected Debian's wireless interface `wlp1s0` to the same Wi-Fi network used by Fedora.

Debian then received:

```
the assigned local Wi-Fi address
```

Fedora was at an address on the local Wi-Fi subnet, so both machines were now on the same the local Wi-Fi subnet subnet. SSH from Fedora to Debian succeeded.

### 2.8 What this troubleshooting taught me

The most important lesson was to test the network in layers instead of treating "internet is broken" as one diagnosis:

1. **Link/interface:** Does the interface exist, and is it up?
2. **Address assignment:** Did DHCP provide an IPv4 address?
3. **Routing/IP reachability:** Can the machine reach a known external IP such as `1.1.1.1`?
4. **DNS:** Can it resolve a hostname?
5. **Package repositories:** Are valid APT sources configured, and can APT fetch their metadata?
6. **Local reachability:** Are the client and server on a network path that allows SSH between them?

The USB tethering workaround got the machine online temporarily, but it did not put Fedora and Debian on the same subnet. The missing `/etc/resolv.conf` explained why IP connectivity worked while domain-based package access failed. Finally, NetworkManager and Wi-Fi restored the intended local network topology.

## 4. System baseline

| Component | Configuration |
|---|---|
| Hostname | `adrian-server` |
| OS | Debian GNU/Linux 13 (trixie) |
| Kernel | `6.12.117+deb13-amd64` |
| Desktop | KDE Plasma 6.3.6, Wayland |
| Display manager | SDDM |
| Network manager | NetworkManager |
| Wi-Fi interface | `wlp1s0` |
| Main user | `serverian` |
| Remote access | OpenSSH + Tailscale |
| Firewall | nftables |

The server currently uses Wi-Fi. Its local address can change, so relying exclusively on its LAN IP would make remote administration unnecessarily fragile.

## 4. Remote access: OpenSSH

### What I configured

- Installed and enabled the OpenSSH server.
- Generated/reused an Ed25519 SSH key on Fedora.
- Installed the public key in the Debian account's `authorized_keys`.
- Verified key-based login from Fedora.
- Applied SSH hardening:
  - Disabled root login over SSH.
  - Disabled password authentication.
  - Disabled keyboard-interactive authentication.
  - Kept public-key authentication enabled.
  - Limited authentication attempts to three.
  - Set a 30-second login grace period.
  - Disabled X11 forwarding.
- Reloaded SSH after validating the configuration with `sshd -t`.
- Tested a fresh SSH connection after the changes.

### Why these choices?

**Public-key authentication** is preferable to password-only SSH because it avoids exposing a reusable account password to remote login attempts. Ed25519 provides modern public-key authentication with small keys and good performance.

**Disabling root SSH login** reduces the impact of direct attacks against the privileged account. Administration should happen through the normal user and narrowly scoped privilege escalation.

**Disabling SSH password authentication** reduces the attack surface exposed to password guessing. This is safe only because key authentication was tested first.

### Verification

A fresh SSH connection from Fedora to Debian succeeded using the key. SSH access was also tested over the Tailscale address.

## 5. Stable remote networking: Tailscale

### What I configured

Installed Tailscale on both Fedora and Debian, enabled the daemon, and authenticated both devices into the same tailnet.

Current addresses recorded during setup:

| Device | Tailscale IPv4 |
|---|---|
| Debian `adrian-server` | assigned dynamically by Tailscale |
| Fedora `fedora-main` | assigned dynamically by Tailscale |

MagicDNS resolved the Debian hostname from Fedora:

```bash
getent hosts adrian-server
```

SSH over Tailscale was tested successfully. The hostname-based connection is intended to be the normal command:

```bash
ssh serverian@adrian-server
```

### Why Tailscale?

The server uses Wi-Fi, and local IP addresses can change when it reconnects or moves between networks. Tailscale gives the devices stable tailnet addresses and hostname-based discovery without requiring router port forwarding.

It also avoids exposing SSH directly to the public internet. This is a practical fit for a personal server that I need to administer remotely.

### Verification

- Both devices appeared connected in `tailscale status`.
- MagicDNS resolved `adrian-server`.
- SSH to the Debian Tailscale address succeeded.

## 6. Firewall: nftables

### What I configured

Installed nftables and enabled its service. The active ruleset uses a default-drop policy for incoming traffic.

The intended policy is:

- Allow loopback traffic.
- Drop invalid connection states.
- Allow established and related traffic.
- Allow ICMP and IPv6 ICMP.
- Allow DHCP replies on the Wi-Fi interface.
- Allow new SSH connections through `tailscale0`.
- Allow SSH from the current local Wi-Fi subnet (the local Wi-Fi subnet) as a fallback.
- Drop unsolicited incoming traffic not explicitly allowed.
- Drop forwarded traffic.
- Allow outbound traffic.

### Why these choices?

A default-deny incoming policy means services are not reachable from the network merely because they happen to be listening. Only explicitly required traffic is allowed.

I kept two SSH paths:

1. Tailscale SSH for normal remote administration.
2. LAN SSH as a fallback when Tailscale is unavailable but the devices are on the same local network.

Outbound traffic remains allowed to avoid breaking ordinary package updates, DNS, and client connections. Forwarding is disabled because the machine is not currently intended to act as a router.

### Verification

The firewall configuration was syntax-checked and applied. The nftables service is enabled and active. A fresh SSH connection over Tailscale succeeded after the Tailscale interface rule was added.

### Important limitation

The LAN fallback is tied to the current subnet, the local Wi-Fi subnet. If the network changes, that rule may need updating. Tailscale access does not depend on that LAN subnet rule.

## 7. Restricted administrative helper

### What I configured

Installed a small administrative helper with:

- A root-owned executable at `/usr/local/sbin/serverian-admin`.
- A user-facing wrapper at `~/.local/bin/serverian-admin`.
- A dedicated sudoers rule allowing the normal user to invoke only the helper without an interactive password.

The helper exposes a limited set of operations rather than a general root shell. Its documented operation categories include package planning/install/removal, package index updates, a specific network wait-online fix, and service actions for an explicit allowlist.

### Why this approach?

Remote automation is useful, but unrestricted passwordless sudo would be a poor security decision. A narrow helper limits what an automation process can do as root.

This is a compromise between operational convenience and least privilege: automate routine, known operations while keeping arbitrary privileged commands out of reach.

### Verification and limitation

The helper's `network-wait-online-fix` operation was executed successfully. It disabled `systemd-networkd-wait-online.service` and reset its failed state. It did **not** disable `systemd-networkd`.

The helper's sudoers validation should be rechecked with `visudo -c` during the next maintenance session, especially before adding more privileged operations.

## 8. Network boot/wait-online issue

During setup, `systemd-networkd-wait-online.service` was failing or delaying startup even though NetworkManager was managing the active Wi-Fi connection.

The machine had both NetworkManager and systemd-networkd services present. The active Wi-Fi connection was managed by NetworkManager, so disabling the networkd *wait-online* unit was more appropriate than disabling the network stack itself.

I disabled only:

```bash
systemd-networkd-wait-online.service
```

NetworkManager and its wait-online service were left enabled. This avoids removing the service actually managing the active Wi-Fi connection.

## 9. KDE configuration migration

### What I migrated

I transferred selected KDE configuration and user customizations from Fedora to Debian, including:

- Global KDE settings and Plasma configuration.
- KWin, input, keyboard shortcut, Dolphin, and Konsole preferences.
- Plasma desktop layout and widgets configuration.
- Custom icon theme: `CRUSHED-Blueberries-Suru`.
- Plasma look-and-feel and desktop theme assets.
- Wallpapers and the animated wallpaper file.
- Custom Plasma widgets, where present.

A backup of Debian's pre-migration configuration was created at:

```
~/kde-pre-migration-backup-<timestamp>
```

The wallpaper path was adjusted from the old Fedora user's home directory to the Debian user's home directory.

A Fedora-specific LookAndFeel package reference was replaced with Debian's Breeze Dark package reference.

### Why migrate instead of rebuilding?

Recreating the desktop manually would take time and risk losing small preferences, shortcuts, layout choices, and customizations. Reusing the existing configuration makes the server's desktop familiar and reduces setup effort.

### Difficulties

- Fedora runs Plasma 6.8.3 while Debian runs Plasma 6.3.6. The version difference can cause unsupported settings or widgets.
- Some widgets or theme components may not exist on Debian.
- The original wallpaper path referenced `the source user home directory`, while the Debian account uses `the Debian user home directory`.
- Fedora's Fedora-specific dark theme package was not appropriate for Debian.

The migration was completed and the configuration files were transferred. A logout/login was recommended so Plasma could reload the settings. Visual compatibility still needs a user-side check.

## 10. Keeping the server awake

### What I configured

Created a persistent user-level systemd service:

```
~/.config/systemd/user/keep-awake.service
```

The service runs:

```bash
/usr/bin/systemd-inhibit \
  --what=sleep:idle \
  --mode=block \
  --who=keep-awake \
  --why="Server must remain available for remote access" \
  /usr/bin/sleep infinity
```

The service is enabled and active. User lingering is enabled, so the user systemd manager can remain running without an interactive login session.

### Why this approach?

The server is intended to provide remote access and background services. Suspending it after inactivity would defeat that purpose.

A systemd sleep/idle inhibitor is preferable to repeatedly simulating keyboard or mouse activity. It expresses the actual requirement directly: do not suspend the machine because it is idle.

### Verification

- `keep-awake.service` is enabled and active.
- `systemd-inhibit --list` shows an active blocking inhibitor for `sleep:idle`.
- `loginctl show-user serverian -p Linger` reports `Linger=yes`.

The display may still turn off or lock. That is separate from system sleep and does not mean the server is offline. The KDE session also has a power-event inhibitor. A physical lid-close test should still be performed rather than assuming every desktop power-management path behaves identically.

## 11. Difficulties encountered and how they were handled

| Difficulty | Cause / context | Resolution |
|---|---|---|
| SSH initially depended on a local IP | Wi-Fi addresses can change | Installed Tailscale and enabled MagicDNS |
| SSH host key warning on first Tailscale connection | The Tailscale address was a new SSH host-key entry on Fedora | Accepted the host key after connecting to the intended server, then verified normal key login |
| SSH hardening risked locking out remote access | Password authentication was being disabled | Tested public-key login before applying the hardening configuration |
| Firewall could have blocked remote administration | Initial rules did not yet include the Tailscale interface | Added an explicit SSH rule for `tailscale0` and tested it |
| Network wait-online failure/delay | systemd-networkd wait-online was not the relevant manager for the active Wi-Fi connection | Disabled only the networkd wait-online unit |
| KDE settings were not fully portable | Different Plasma versions and Fedora-specific theme references | Adjusted the wallpaper path and LookAndFeel reference; retained a pre-migration backup |
| Initial keep-awake service failed | The first systemd-inhibit command did not provide a command to run | Changed it to run `/usr/bin/sleep infinity` under the inhibitor and verified the active block |

## 12. Current state

| Area | Status | Evidence |
|---|---|---|
| Installation networking | Recovered | USB tethering provided temporary IP connectivity; DNS was restored by creating resolv.conf; Wi-Fi was then configured through NetworkManager |
| Debian boot and services | Operational | NetworkManager, SSH and nftables active |
| SSH key login | Tested | Fresh connection from Fedora succeeded |
| SSH hardening | Applied | Password and root SSH login disabled in the staged hardening configuration |
| Tailscale | Tested | Both devices connected; MagicDNS resolved |
| nftables | Active | Default-drop incoming ruleset with Tailscale SSH exception |
| Restricted admin helper | Installed | Network wait-online operation executed |
| KDE migration | Transferred | Backup exists; visual verification still needed |
| Keep-awake | Active | Persistent user service and sleep/idle inhibitor verified |

## 13. Future development roadmap

The following items are planned improvements, not features currently claimed as implemented. They are ordered by operational value: first make the server recoverable, then tighten and verify its security posture, and finally improve observability and repeatability.

### Phase 1: Reliability and recovery

**Objective:** Ensure the server can be maintained and restored after routine failures.

Planned work:
- Configure unattended security updates and define a controlled reboot policy.
- Implement scheduled backups for important data and system configuration, including at least one copy stored off-server.
- Document recovery procedures for network misconfiguration and failed boots.
- Perform a full reboot test and verify that Tailscale connectivity and SSH access return automatically.
- Test lid-close and extended-idle behavior under the actual hardware and desktop power-management configuration.

**Completion criteria:**
- Update behavior and reboot conditions are documented and tested.
- A backup can be restored from the off-server copy.
- Recovery steps are documented and have been exercised.
- Remote access is verified after reboot and after an extended idle period.

### Phase 2: Security validation and hardening

**Objective:** Verify that the current access controls work as intended and that only required services are exposed.

Planned work:
- Validate sudoers syntax with `sudo visudo -c` and review the restricted administrative helper's permissions and command allowlist.
- Audit listening sockets with `ss -tulpn`; document each network-facing service and its purpose.
- Review Tailscale device authorization, account security, and key-expiry settings.
- Reassess whether the local-subnet SSH fallback is necessary.
- Establish a periodic review of authentication and system logs using `journalctl` and `journalctl -u ssh`.

**Completion criteria:**
- Sudoers configuration passes validation.
- Every listening network service has a documented purpose and an intentional firewall policy.
- Tailscale access and device authorization settings have been reviewed.
- The decision to retain or remove the LAN SSH fallback is documented.
- A repeatable log-review procedure exists.

### Phase 3: Monitoring and maintenance

**Objective:** Detect resource pressure and service failures before they interrupt server availability.

Planned work:
- Select and deploy lightweight monitoring for CPU, memory, storage capacity, temperature, and network availability.
- Configure disk-space thresholds and alerts.
- Track systemd service health and failed units.
- Document a maintenance procedure covering updates, service checks, and rollback considerations.
- Evaluate SMART monitoring based on the storage hardware exposed to the operating system.

**Completion criteria:**
- Key resource metrics can be inspected without manual log collection.
- Disk-space warnings trigger at documented thresholds.
- Service failures are visible through a documented check or notification.
- Update and rollback procedures have been tested or clearly state their limitations.

### Phase 4: Configuration management and documentation

**Objective:** Make the server configuration understandable, auditable, and reproducible.

Planned work:
- Maintain a network diagram and an inventory of exposed services.
- Document the administrative helper's exact allowlist, privilege boundary, and safe extension process.
- Track non-secret configuration files in version control. Private keys, credentials, tokens, and other secrets must remain outside the repository.
- Once the manual setup is stable, evaluate an Ansible playbook or equivalent automation for repeatable provisioning.
- Maintain a changelog describing significant configuration changes and their rationale.

**Completion criteria:**
- Network topology and service exposure are documented and kept current.
- Administrative privileges and helper behavior are documented.
- Configuration history is version-controlled without secrets.
- A clean rebuild or equivalent provisioning test can reproduce the documented baseline.
- Significant changes are recorded in the changelog.

**Scope note:** These phases are a roadmap for future work. The current implementation status is described in Section 12; planned controls should only be marked complete after implementation and verification.

## 14. Reflection

The main design principle across this work was to make the server reachable and maintainable without making it unnecessarily exposed. The initial three-hour networking problem also demonstrated why layered diagnosis matters: link state, DHCP, routing, DNS, repository configuration, and subnet reachability are separate failure points.

Tailscale solves the changing-address problem. SSH keys and disabled password login reduce the remote authentication attack surface. nftables limits unsolicited incoming traffic. The restricted administrative helper avoids granting automation unrestricted root access. The keep-awake service makes the machine suitable for unattended use.

The next stage should not be adding random services just because the machine is now reachable. It should be validating recovery, backups, updates, monitoring, and the actual workload the server is meant to host.

---

## Appendix: Useful commands

### Connect from Fedora

```bash
ssh serverian@adrian-server
```

### Check Tailscale

```bash
tailscale status
tailscale ip -4
```

### Check SSH

```bash
sudo systemctl status ssh
sudo sshd -t
```

### Check firewall

```bash
sudo nft list ruleset
sudo systemctl status nftables
```

### Check keep-awake

```systemd
systemctl --user status keep-awake.service
systemd-inhibit --list
loginctl show-user serverian -p Linger
```

### Check failed services and recent logs

```bash
systemctl --failed
journalctl -p warning -b
```
