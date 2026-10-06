---
layout: default
title: Baseline Hardening for a Small Linux Host
date: 2026-10-06 12:00:00 +0000
categories: hardening linux
---

Small labs and single-purpose servers fail in predictable ways: exposed SSH, leftover packages, and logging nobody reads. This note is the baseline we apply before anything interesting runs on a fresh Linux host.

## Scope

This checklist targets a single Ubuntu or Debian VPS used for experiments, reverse proxies, or lightweight services. It is not a compliance program. It is the minimum we want in place before opening the box to the internet.

## 1. Inventory and reduce the attack surface

Before tuning configs, know what is listening:

```bash
ss -tulpn
systemctl list-units --type=service --state=running
```

Remove packages and services you do not need. Disable unused daemons rather than leaving them “for later.” Every open port is a promise you have to keep maintaining.

## 2. Accounts and privilege

- Prefer SSH keys over passwords; disable password authentication once keys work.
- Create a non-root admin user and use `sudo` for elevation.
- Lock or remove unused accounts.
- Keep `PermitRootLogin no` in `sshd_config` after your admin user is verified.

Example `sshd_config` fragments we standardize on:

```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
X11Forwarding no
AllowUsers youradmin
```

Reload SSH only after confirming a second session still authenticates with the new settings.

## 3. Updates as a habit, not an event

Unattended security updates catch the quiet CVEs that do not make headlines:

```bash
sudo apt update
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

Pin a calendar reminder for kernel reboots. Patches that never get applied after a reboot are theater.

## 4. Firewall: default deny inbound

UFW is enough for most single-host labs:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status verbose
```

If you terminate TLS on the same host, allow only the ports you intentionally publish (typically 80/443). Everything else stays closed.

## 5. Fail2ban for noisy auth attempts

Credential stuffing against SSH is background noise on the public internet. Fail2ban reduces the volume:

```bash
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban
```

Start with the default SSH jail. Tune ban times later once you have a week of logs.

## 6. Time, logs, and persistence

- Sync clocks with NTP (systemd-timesyncd is fine).
- Confirm `journald` retains enough history for incident review.
- Ship critical logs off-box when the host matters; local-only logs vanish with the disk.

A host you cannot investigate after compromise is a host you cannot learn from.

## 7. Backups before experiments

Treat the first successful backup restore as part of provisioning, not an afterthought:

1. Snapshot or copy `/etc`, application data, and any secrets store.
2. Restore once to a scratch host or directory.
3. Document the restore command in the same repo as the automation that built the box.

## What we skip on purpose (for now)

Full disk encryption, centralized SIEM, and elaborate IDS belong in later notes. A small host that is patched, firewalled, key-only, and recoverable already outruns most opportunistic scanning.

## Closing

Security for small infrastructure is mostly boring discipline. Inventory, reduce, authenticate carefully, patch, deny by default, and prove you can restore. We will use this baseline as the starting point for future lab entries on VPN access patterns and service-specific hardening.
