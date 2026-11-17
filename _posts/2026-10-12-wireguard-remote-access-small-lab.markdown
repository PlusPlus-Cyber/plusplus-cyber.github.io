---
layout: default
title: WireGuard Remote Access for a Small Lab
date: 2026-10-12 13:00:00 +0000
categories: vpn networking
---

Port-forwarded SSH and ad-hoc “open this for a minute” firewall holes are how small labs get noisy. This note is how we put a single WireGuard peer mesh in front of admin access so the rest of the host stays closed to the public internet.

## Scope

One Ubuntu or Debian VPS, a handful of laptops or phones that need admin access, and no requirement for a full mesh product. We are not covering site-to-site tunnels between offices here—just remote access into a lab host that already has the [baseline hardening](/2026/10/06/baseline-hardening-small-linux-host/) checklist applied.

## Why WireGuard for this job

- Small config surface: keys, allowed IPs, listen port.
- Runs as a normal interface (`wg0`) you can firewall and log like anything else.
- Easy to revoke: remove a peer block and reload.

Treat it as a private admin network, not a general-purpose internet gateway, unless you deliberately route full traffic through it.

## 1. Install and generate keys on the server

```bash
sudo apt update
sudo apt install -y wireguard
umask 077
wg genkey | tee /etc/wireguard/server.key | wg pubkey > /etc/wireguard/server.pub
```

Keep private keys mode `600` and never commit them to this repo.

## 2. Minimal server config

Create `/etc/wireguard/wg0.conf`:

```ini
[Interface]
Address = 10.67.0.1/24
ListenPort = 51820
PrivateKey = <server private key>
# Optional: NAT if clients should exit through this host
# PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
# PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

# Example admin laptop peer — add one block per device
[Peer]
PublicKey = <client public key>
AllowedIPs = 10.67.0.2/32
```

Enable and start:

```bash
sudo systemctl enable --now wg-quick@wg0
sudo wg show
```

## 3. Client peer

On each admin device, generate a keypair the same way, then configure:

```ini
[Interface]
PrivateKey = <client private key>
Address = 10.67.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = <server public key>
Endpoint = your.lab.example:51820
AllowedIPs = 10.67.0.0/24
PersistentKeepalive = 25
```

`AllowedIPs = 10.67.0.0/24` keeps the tunnel scoped to the lab network. Use a broader list only when you intentionally want all traffic to hairpin through the VPS.

## 4. Firewall and SSH posture

On the host:

```bash
sudo ufw allow 51820/udp comment 'wireguard'
sudo ufw allow from 10.67.0.0/24 to any port 22 proto tcp comment 'ssh via wg'
sudo ufw status verbose
```

Once every admin reaches the host over WireGuard:

1. Confirm SSH works via `10.67.0.1`.
2. Remove public SSH (`22/tcp`) from UFW if nothing else needs it.
3. Keep the earlier key-only SSH settings; the VPN is an extra gate, not a password substitute.

## 5. Operational habits

- One peer per device. Shared keys make revocation guesswork.
- Name peers in comments (`# alice-laptop`, `# oncall-phone`).
- Rotate a peer by generating a new keypair and replacing its `[Peer]` block; reload with `sudo systemctl reload wg-quick@wg0` (or bounce the interface if your distro reload path is unreliable).
- Monitor with `sudo wg show`—stale handshakes are often dead clients or bad endpoints.

## 6. What we deliberately skip

Split-tunnel policy engines, MDM-enforced profiles, and dual-homed HA gateways belong in later notes. A single `wg0` with per-device peers and SSH locked to the tunnel already removes the common failure mode of “SSH open to the world because someone needed to get in from the airport.”

## Closing

WireGuard is useful when it shrinks what you expose, not when it becomes another always-on appliance nobody documents. Pair it with the baseline host checklist, keep peer inventory boring and current, and treat the tunnel address as the only path for admin work.
