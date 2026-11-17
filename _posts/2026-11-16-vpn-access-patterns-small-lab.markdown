---
layout: default
title: VPN Access Patterns for a Small Lab
date: 2026-11-16 18:00:00 -0600
categories: vpn networking linux
---

Once a host is patched, firewalled, and key-only, the next failure mode is usually remote access. Exposing SSH, admin panels, or home-lab services to the whole internet is convenient until it is not. This note covers the VPN patterns we use so management traffic never needs a public listener.

## Scope

This is for a small lab or single VPS: a handful of services, one or two operators, and a desire to keep administrative ports off the public internet. It is not a full zero-trust platform design. Pair it with the [baseline hardening checklist](/2026/10/06/baseline-hardening-small-linux-host.html) before you open anything new.

## Why VPN first

A public SSH port with keys and fail2ban is better than passwords. A host that only accepts management traffic over a VPN is better still:

- Auth failures never hit your real SSH daemon from the open internet.
- Broken experiments stay reachable without opening temporary holes “just for today.”
- You get a clear boundary: public services on 80/443; everything else on the tunnel.

Treat the VPN as the front door for humans and automation. Treat public ports as the shop window.

## Pattern 1: Operator tunnel into a private network

Use when one or more lab hosts should not expose SSH at all.

1. Run a WireGuard (or similar) endpoint on a hardened edge host.
2. Put lab hosts on a private address range reachable only through that tunnel.
3. Bind SSH and admin UIs to private interfaces or VPN addresses.
4. Keep UFW (or equivalent) default-deny on the public side; allow only the VPN UDP port plus intentional public services.

Minimal mental model:

```text
Operator laptop  --WireGuard-->  edge host  --private net-->  lab VMs
                                      |
                                   public 443 (optional apps)
```

Prefer a single edge entry point over VPN listeners on every box. Fewer public surfaces means fewer certificates, fewer UDP holes, and one place to rotate keys.

## Pattern 2: Split tunnel for admin only

Full-tunnel VPN that sends all traffic through the lab is rarely what a small team needs. Prefer split tunnel:

- Route only lab subnets (for example `10.10.0.0/24`) through the tunnel.
- Leave general web browsing on the operator’s normal path.
- Document the routes in the same place you store the peer configs.

Split tunnel reduces blast radius when a lab host is compromised and keeps latency predictable for day-to-day work.

## Pattern 3: Jump host, not a second internet

If some cloud VMs must stay on a provider network, use the VPN edge as a jump host rather than publishing SSH everywhere:

```bash
# After the WireGuard interface is up and routes exist:
ssh -J admin@edge.lab.internal admin@app-01.lab.internal
```

Or configure `ProxyJump` in `~/.ssh/config` so muscle memory stays simple. The public internet should see the VPN endpoint, not a constellation of SSH banners.

## Key and peer hygiene

VPN misconfiguration ages badly. Keep the same discipline you use for SSH keys:

- One peer identity per person and per automation principal; no shared laptop keys.
- Record `AllowedIPs`, endpoint, and last-rotated date next to the public key.
- Remove peers the day access ends; dormant peers are unpaid technical debt.
- Prefer short-lived or easily rotated keys for contractors and temporary builds.

A peer list you cannot explain is an access-control list you do not have.

## Firewall rules that match the story

On the edge host, make the policy readable:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 51820/udp comment 'wireguard'
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
# Do not allow 22/tcp from the world once VPN access is verified
sudo ufw status verbose
```

Verify SSH still works over the tunnel from a second device before you drop public port 22. Losing your only path in is an avoidable outage.

## What we measure

After the tunnel is live, confirm the boring truths:

- `ss -tulpn` on lab hosts shows SSH bound to private or VPN addresses only.
- Failed auth attempts against public SSH drop to near zero (or the port is closed entirely).
- A peer revoke actually stops connectivity within one config reload.
- Backup and restore notes still work when the operator is on the VPN path.

## What we skip on purpose (for now)

IdP-backed device posture, per-app tunnels, and full mesh overlays belong in later notes. For a small lab, a single WireGuard edge, private SSH, split routes, and a maintained peer list already remove most casual exposure.

## Closing

VPN access is not a product feature; it is a default path for management. Put operators on a tunnel, keep admin ports private, rotate peers like credentials, and prove you can still get in after the public SSH door closes. Next lab entries will dig into service-specific hardening once that access pattern is assumed.
