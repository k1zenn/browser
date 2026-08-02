# Tailscale Exit Node Guide — Bypass College WiFi Game Blocking

## Why this works

The college firewall (Palo Alto PAN-OS) blocks Steam/game traffic, and Cloudflare WARP only
works in "DNS only" mode because the firewall drops WARP's WireGuard UDP traffic. DNS-only
tunneling doesn't help Steam, which needs a full traffic tunnel.

Solution: route all your laptop traffic through an Azure VM (exit node) via Tailscale.
If the firewall blocks direct WireGuard, Tailscale automatically falls back to DERP relays
over TCP 443 (normal HTTPS), which the firewall allows.

## What you need

- An Azure VM (Debian-based) — any small/cheap tier works
- SSH access to the VM
- Tailscale account (free)

## Step 1: Install Tailscale on the Azure VM

SSH into the VM:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --advertise-exit-node
```

This connects the VM to your tailnet and advertises it as an exit node.

## Step 2: Approve the exit node in the admin console

1. Go to https://console.tailscale.com/admin/machines
2. Find the Azure VM
3. Toggle **"Use as exit node"** on (required once)

## Step 3: Enable IP forwarding on the VM

```bash
cat /proc/sys/net/ipv4/ip_forward
```

If it shows `0` (or if `sysctl` is not found — use the `/proc` path above):

```bash
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

## Step 4: Add the NAT rule (MASQUERADE)

```bash
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

Verify:

```bash
sudo iptables -t nat -L POSTROUTING
```

You should see a MASQUERADE line.

Note: `--advertise-exit-node` may enable forwarding automatically, but the MASQUERADE
rule is the one that was missing here — without it, traffic is routed to the VM and
dropped ("black hole").

## Step 5: Make the rules survive a reboot

```bash
sudo apt install -y iptables-persistent
```

Answer **Yes** to "Save current IPv4 rules?" and **No** to the IPv6 question.

Then save manually after any future rule changes:

```bash
sudo netfilter-persistent save
```

## Step 6: Use the exit node on your laptop (Windows)

Install Tailscale on the laptop and sign in to the same account.

GUI: right-click the Tailscale tray icon → **Exit Node** → select the Azure VM.

Or command line:

```powershell
tailscale up --exit-node=<VM-hostname>
```

Use the machine **name** shown in the admin console (not the Azure public IP).

## Verify it works

```bash
curl ifconfig.me
```

You should see the Azure IP, not your college IP. Also try speedtest.net — it should
report the Azure location.

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `curl ifconfig.me` times out after selecting exit node | IP forwarding off (Step 3) or MASQUERADE missing (Step 4) |
| Exit node not selectable | Not approved in admin console (Step 2) |
| Works but slow | You're on a DERP relay (TCP 443); direct WireGuard usually connects later |

## Steam notes

- If Steam client itself struggles, add `-tcp` to its launch options.
- Gameplay traffic (UDP 27000-27050) is handled by the tunnel automatically.

## Honest warning

This is network policy circumvention. Your college's acceptable use policy probably
forbids it, and the firewall admins can block this too (e.g. by app-ID rules or
blocking Tailscale's DERP endpoints). Use it responsibly and at your own risk.
