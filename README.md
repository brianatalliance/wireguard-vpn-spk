# WireGuard VPN Tunnel SPK

**Author:** Brian Vicente
**Version:** 1.0.0
**Date:** 2026-03-28
**Org:** Alliance for Empowerment

Synology SPK package that provides a userspace WireGuard VPN tunnel for DSM 7.0+ without requiring kernel module support. Uses `wireguard-go` (Go implementation) with `wg` and `wg-quick` from `wireguard-tools`.

## Target Platform

| Spec | Value |
|------|-------|
| NAS Model | Synology DS220+ |
| Architecture | x86_64 (Geminilake) |
| DSM Version | 7.0+ |
| Privilege | `run-as: package` (DSM 7 compatible) |

## What's Included

| Component | Description |
|-----------|-------------|
| `wireguard-go` | Userspace WireGuard implementation (statically compiled from [source](https://git.zx2c4.com/wireguard-go)) |
| `wg` | WireGuard CLI tool (statically compiled from [wireguard-tools](https://git.zx2c4.com/wireguard-tools)) |
| `wg-quick` | Standard tunnel bring-up/down script |
| `wg-tunnel-setup` | Helper script for interactive config, status, testing, and reset |
| `VERSION` | Version tracking and changelog |

## Building the SPK

### Prerequisites

- Go 1.22+ (for compiling `wireguard-go`)
- GCC (for compiling `wg`)
- `tar` (for packaging)

### Build Steps

```bash
# 1. Compile wireguard-go (static)
git clone https://git.zx2c4.com/wireguard-go /tmp/wireguard-go
cd /tmp/wireguard-go
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags="-s -w" -o package/wireguard-go .

# 2. Compile wg (static)
git clone https://git.zx2c4.com/wireguard-tools /tmp/wireguard-tools
cd /tmp/wireguard-tools/src
make CC="gcc" LDFLAGS="-static"
cp wg package/wg
strip package/wg

# 3. Copy wg-quick
cp /tmp/wireguard-tools/src/wg-quick/linux.bash package/wg-quick

# 4. Create package.tgz
cd package && tar czf ../package.tgz wireguard-go wg wg-quick wg-tunnel-setup VERSION && cd ..

# 5. Build SPK
tar cf WireGuardVPN-geminilake-1.0.0.spk INFO package.tgz scripts conf PACKAGE_ICON.PNG PACKAGE_ICON_256.PNG LICENSE
```

## Installation

1. Open DSM → **Package Center** → **Manual Install**
2. Upload `WireGuardVPN-geminilake-1.0.0.spk`
3. **Uncheck** "Run after installation"
4. After install, SSH into the NAS and configure:

```bash
# Edit config with your VPN credentials
sudo vi /etc/wireguard/wg0.conf

# Start the tunnel (requires root)
sudo /var/packages/WireGuardVPN/scripts/start-stop-status start

# Verify
sudo /var/packages/WireGuardVPN/target/wg show wg0
curl -s https://ifconfig.me
```

## Configuration

Edit `/etc/wireguard/wg0.conf`:

```ini
[Interface]
PrivateKey = <YOUR_PRIVATE_KEY>
Address = <YOUR_VPN_IP>/32
# DNS line removed — DSM lacks resolvconf

[Peer]
PublicKey = <PEER_PUBLIC_KEY>
AllowedIPs = 0.0.0.0/0
Endpoint = <SERVER_IP>:<PORT>
PersistentKeepalive = 25
```

## Helper Tool

```bash
wg-tunnel-setup status     # Show tunnel status + public IP
wg-tunnel-setup test       # Run connectivity diagnostics
wg-tunnel-setup configure  # Interactive config wizard
wg-tunnel-setup reset      # Stop and clean up
```

## DSM 7 Notes

- DSM 7 blocks unsigned third-party packages from running as root
- Package uses `run-as: package` privilege level for GUI installation
- Start/stop requires `sudo` via SSH
- For auto-start on boot, create a Triggered Task in DSM Task Scheduler (Boot-up, root)

## Architecture Note

For production use, routing through a **UDM Pro WireGuard VPN Client** is recommended over running WireGuard directly on the NAS. The UDM Pro handles the tunnel at the gateway level, and the NAS simply routes through it — no kernel modules, no DSM compatibility issues, survives NAS reboots and DSM updates cleanly.

## Project Structure

```
wireguard-spk/
├── INFO                    # SPK metadata
├── LICENSE                 # MIT + bundled component licenses
├── PACKAGE_ICON.PNG        # 64x64 icon for Package Center
├── PACKAGE_ICON_256.PNG    # 256x256 icon for Package Center
├── conf/
│   ├── privilege           # run-as: package (DSM 7 compatible)
│   └── resource            # usr-local-linker for PATH
├── package/
│   ├── wireguard-go        # (built from source, not in repo)
│   ├── wg                  # (built from source, not in repo)
│   ├── wg-quick            # Bash script from wireguard-tools
│   ├── wg-tunnel-setup     # Helper script
│   └── VERSION             # Version tracking
└── scripts/
    ├── postinst            # Post-install: symlinks, config dir, template
    ├── postuninst          # Post-uninstall: cleanup symlinks
    ├── postupgrade         # Post-upgrade: re-link binaries
    ├── preinst             # Pre-install: ensure TUN device
    ├── preuninst           # Pre-uninstall: stop tunnel
    ├── preupgrade          # Pre-upgrade: stop tunnel
    └── start-stop-status   # Main lifecycle script
```

## Related Projects

- [synology-connector](https://github.com/brianatalliance/synology-connector) — Synology DSM API connector
- [udm-nspawn-pki](https://github.com/brianatalliance/udm-nspawn-pki) — UDM Pro PKI automation
- [atera-connector](https://github.com/brianatalliance/atera-connector) — Atera RMM API connector
- [atera-dashboard](https://github.com/brianatalliance/atera-dashboard) — NOC dashboard

## License

MIT — see [LICENSE](LICENSE) for bundled component licenses.

WireGuard is a registered trademark of Jason A. Donenfeld.
