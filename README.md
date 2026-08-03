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

## Verify it survived a reboot

Reboot the VM (`sudo reboot`), then SSH back in and check:

```bash
sudo iptables -t nat -L POSTROUTING
```

You should still see the MASQUERADE line after reboot — that proves it persisted.

Then on the laptop:

```bash
curl ifconfig.me
```

You should still see the Azure IP.

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

- If Steam client itself fails, add `-tcp` to its launch options.
- Gameplay traffic (UDP 27000-27050) is handled by the tunnel automatically.

---

# Faster option: Xray VLESS+Reality tunnel (no relay)

Tailscale on college WiFi is forced onto a DERP relay (TCP 443), which caps you at
roughly 6 Mbps. If you want your real speed back, run a direct tunnel from your laptop
to the Azure VM over TCP 443 — one hop, no middleman, looks like normal HTTPS to the
firewall.

Route: laptop → (TCP 443, direct) → Azure VM → internet

## Server: install Xray on the Azure VM

```bash
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```

## Generate your keys and IDs

```bash
xray uuid                                # UUID (client id)
xray x25519                              # PrivateKey + PublicKey
openssl rand -hex 8                      # shortId (16 hex chars)
```

Note: if `xray` isn't on your PATH, run the binaries from `/usr/local/xray/`.

## Server config — `/usr/local/etc/xray/config.json`

Replace the placeholders (UUID, PrivateKey, shortId) with your generated values:

```json
{
  "log": { "loglevel": "warning" },
  "inbounds": [
    {
      "listen": "0.0.0.0",
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [
          { "id": "REPLACE_UUID", "flow": "xtls-rprx-vision" }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "www.microsoft.com:443",
          "serverNames": ["www.microsoft.com"],
          "privateKey": "REPLACE_PRIVATE_KEY",
          "shortIds": ["REPLACE_SHORTID"]
        }
      }
    }
  ],
  "outbounds": [
    { "protocol": "freedom" },
    { "protocol": "blackhole" }
  ]
}
```

## Apply the config

```bash
sudo systemctl restart xray
sudo systemctl enable xray
sudo systemctl status xray        # confirm "active (running)"
```

## Open port 443 in Azure

In the Azure portal go to the VM → **Networking → Inbound port rules** → add:
TCP **443**, destination `*`, source `Internet`, action `Allow`.

## Client: v2rayN on Windows

1. Download **v2rayN** from https://github.com/2dust/v2rayN/releases and unzip.
2. Edit the `config.json` inside v2rayN's folder with your REAL Azure config
   (the same one from above) and start it.
3. Server → Add; or edit `config.json`:
   - Address: your Azure VM **public IP**
   - Port: `443`
   - UUID: `REPLACE_UUID`
   - Flow: `xtls-rprx-vision`
   - Encryption: `none`
   - Network: `tcp`
   - Security: `reality`
   - Fingerprint: `chrome`
   - SNI / ServerName: `www.microsoft.com`
   - PublicKey: `REPLACE_PUBLIC_KEY`
   - ShortID: `REPLACE_SHORTID`
4. Select the server, then enable **TUN mode** in v2rayN so ALL traffic (including
   Steam/game UDP) goes through the tunnel.

## Verify

```powershell
curl ifconfig.me
```

Should show the Azure IP — at real throughput this time.

## Why Reality instead of a plain TLS cert

Reality uses no certificate at all: it fronts a real website (Microsoft) so even
DPI firewalls see an ordinary HTTPS connection to microsoft.com. More robust than
TLS with a self-signed cert, and no domain needed.

## Optional: enable BBR for better throughput

```bash
echo -e 'net.core.default_qdisc=fq\nnet.ipv4.tcp_congestion_control=bbr' | sudo tee -a /etc/sysctl.d/99-tailscale-exit.conf
sudo sysctl --system
```

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Connection refused on 443 | Open port 443 in Azure NSG; confirm `systemctl status xray` active |
| Handshake fails / user error | Check `journalctl -u xray -n 50`; UUID/keys/shortId must match client exactly |
| Still slow | Full path is now direct TCP; check VM's Azure outbound bandwidth |

## Honest warning

This is network policy circumvention. Your college's acceptable use policy probably
forbids it, and the firewall admins can block this too (e.g. by app-ID rules or
blocking Tailscale's DERP endpoints). Use it responsibly and at your own risk.
