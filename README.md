# MicroLink v2 — ESP32 Tailscale Client

Production-ready Tailscale VPN client for the ESP32 platform with WiFi and 4G cellular support. Includes a fully **authenticated web config server** with **first-boot SoftAP provisioning**, so the device can be onboarded onto a new network entirely through a browser.

Should work on most ESP32 variants (ESP32, ESP32-S3, ESP32-P4, etc.) — ESP32-S3 with PSRAM recommended for production.

## Features

- **Full Tailscale Protocol Support**
  - ts2021 coordination protocol
  - WireGuard encryption (ChaCha20-Poly1305)
  - DISCO path discovery (PING/PONG/CALL_ME_MAYBE)
  - DERP relay with dynamic region discovery (up to 32 regions)
  - STUN for public IP / NAT type discovery (IPv4 + IPv6)
  - Delta updates (PeersChanged, PeersRemoved, PeersChangedPatch)
  - MagicDNS hostname resolution (short name or FQDN)
  - Key expiry detection and auto re-registration

- **WiFi + 4G Cellular**
  - WiFi primary with automatic cellular failback
  - PPP cellular data — real lwIP sockets, direct UDP, NAT hole-punching
  - AT socket bridge fallback for carriers that reject PPP auth
  - Multi-carrier: PAP (IMSI-based) and CHAP (credential-based) automatic selection
  - Seamless network rebind — switch between WiFi and cellular without destroying the VPN session (~330ms rebind, ~7s recovery)
  - Network health monitoring with automatic failback to WiFi when recovered

- **Subnet Router Mode**
  - Advertise LAN subnets to the tailnet (IP_FORWARD + IPv4 NAPT)
  - Routes configurable from the web UI and persisted in NVS — no reflash needed

- **Authenticated Web Config Server**
  - HTTP Basic auth on every endpoint (login required — `admin` + password)
  - Admin password stored in NVS, changeable via the UI, seeded from Kconfig; **fail-closed** when empty
  - System monitor: temperature, WiFi RSSI, uptime, heap/PSRAM, DERP region, peer count, task stack watermarks
  - WiFi management: multi-SSID list with priority ordering, **on-device AP scan + one-tap connect**, live connection status
  - Subnet routes (advertise_routes) visual editing — persisted to NVS, applied on restart
  - Peer allowlist for DISCO probe filtering (immediate effect)
  - Device settings (auth key, device name, cellular, PPP, control plane host, debug flags)
  - All settings persist in NVS — no rebuild needed

- **First-Boot / No-WiFi Provisioning**
  - If no WiFi is reachable (fresh device or new environment), the ESP32 opens its own open SoftAP `MicroLink-Config`
  - Connect a phone/laptop to it, open `http://192.168.4.1`, and configure WiFi in the browser — no serial, no reflash

- **Production Ready**
  - Fully async, task-based architecture (no polling loop)
  - Tested with 300+ peer tailnets (PSRAM-backed 512KB buffers)
  - NVS peer cache — DISCO probing starts immediately on reboot
  - Proactive H2 WINDOW_UPDATE for fast MapResponse downloads
  - Zero-copy WireGuard receive (raw lwIP PCB, for 30fps+ video streaming)
  - DISCO peer filtering / allowlist, priority peer, Headscale / Ionscale compatible
  - Credential security: all secrets in git-ignored `sdkconfig`

## Requirements

- ESP-IDF v5.0 or later (tested with v5.3 and v6.1)
- ESP32 with WiFi (ESP32-S3 with PSRAM recommended)
- Tailscale account with auth key (generate at https://login.tailscale.com/admin/settings/keys)
- For cellular: ESP32-compatible 4G cellular module (e.g., SIM7600, SIM7670) + active SIM card

## Hardware

### Tested Boards

| Board | Type | Notes |
|-------|------|-------|
| ESP32-S3 with 8MB PSRAM | WiFi | Recommended for production |
| Seeed Studio XIAO ESP32S3 | WiFi + Cellular | Pairs with Waveshare SIM7600X |
| Waveshare ESP32-S3-Touch-AMOLED-2.06 | WiFi | Touchscreen display |
| HiLetgo ESP-32S | WiFi | Budget option, no PSRAM |
| ESP32-WROOM-32D / DevKitC | WiFi | Standard dev board, no PSRAM |

### Tested Cellular Modules

| Module | Interface | Notes |
|--------|-----------|-------|
| Waveshare SIM7600G-H 4G | UART | PPP + AT socket bridge |
| LILYGO T-SIM7670G-S3 | UART | Integrated ESP32-S3 + SIM7670G |

### Should Work (Untested)

MicroLink uses standard ESP-IDF APIs — any ESP32 variant with WiFi and sufficient RAM should work. Boards with PSRAM are recommended for large tailnets (100+ peers).

## Quick Start

### 1. Clone and enter an example

```bash
git clone https://github.com/mdreamfly/MicroLink_plus.git
cd microlink/examples/basic_connect    # or: cellular_connect, cellular_heartbeat, failover_connect
```

### 2. Configure sdkconfig

Add these settings to your `sdkconfig.defaults` file:

```ini
# PSRAM Configuration (required for ESP32-S3 with PSRAM)
CONFIG_SPIRAM=y
CONFIG_SPIRAM_MODE_OCT=y
CONFIG_SPIRAM_TYPE_AUTO=y
CONFIG_SPIRAM_SPEED_80M=y
CONFIG_SPIRAM_ALLOW_STACK_EXTERNAL_MEMORY=y
CONFIG_SPIRAM_MALLOC_ALWAYSINTERNAL=4096
CONFIG_SPIRAM_MALLOC_RESERVE_INTERNAL=32768

# Partition table (app needs ~1MB+)
CONFIG_PARTITION_TABLE_SINGLE_APP_LARGE=y

# TLS/HTTPS (required for DERP and control plane)
CONFIG_ESP_TLS_USING_MBEDTLS=y
CONFIG_MBEDTLS_SSL_PROTO_TLS1_2=y
CONFIG_MBEDTLS_CERTIFICATE_BUNDLE=y
CONFIG_MBEDTLS_CERTIFICATE_BUNDLE_DEFAULT_CMN=y

# Subnet router (optional): forward + NAPT LAN traffic
CONFIG_LWIP_IP_FORWARD=y
CONFIG_LWIP_IPV4_NAPT=y

# Networking
CONFIG_LWIP_IPV4=y
CONFIG_LWIP_IP4_FRAG=y
CONFIG_LWIP_IP4_REASSEMBLY=y

# Stack size
CONFIG_ESP_MAIN_TASK_STACK_SIZE=8192
```

### 3. Configure credentials

```bash
cp sdkconfig.credentials.example sdkconfig.credentials
# Edit sdkconfig.credentials with your WiFi SSID/password, Tailscale auth key, etc.
```

Or run `idf.py menuconfig` → MicroLink V2 → Credentials to set them interactively.

Credentials are stored in `sdkconfig` (which is gitignored) so they are never accidentally committed to version control.

### 4. Build and flash

```bash
source ~/esp/esp-idf/export.sh
idf.py build
idf.py -p /dev/ttyACM0 flash monitor
```

### 5. Test

From any device on your tailnet:

```bash
tailscale ping esp32-microlink
```

You should see:

```
pong from esp32-microlink (100.x.x.x) via DERP(dfw) in 150ms
```

Then open the config UI in a browser: `http://<vpn-ip>/` and log in with `admin` / your admin password (default seeded from `CONFIG_ML_CONFIG_PASSWORD`, see below).

## First Boot — No-WiFi Provisioning

If the device cannot reach any configured WiFi — a fresh unit with nothing configured, or a device moved to a new environment — it falls back to **SoftAP provisioning mode**:

1. The ESP32 opens an open WiFi AP named **`MicroLink-Config`** (max 4 clients).
2. Connect your phone or laptop to that AP.
3. Open **`http://192.168.4.1`** in a browser and log in (`admin` + password).
4. In **WiFi Networks**, add the new environment's SSID/password (use **Scan** to list nearby APs, or type it manually).
5. Press **Restart Device**. The device reboots, joins the new network, and runs normally.

Notes:
- The AP is open (no WiFi password), but the config page still requires the admin login.
- With no WiFi configured at all, the AP starts immediately; if WiFi is configured but unreachable, it starts after a ~45s timeout.
- While connected to a network normally, no SoftAP is broadcast.

## Memory Footprint

### Static Memory (measured with `idf.py size`, ESP-IDF v5.3, ESP32-S3)

| Build | SRAM (static) | Flash | IRAM | Free SRAM |
|-------|---------------|-------|------|-----------|
| WiFi + Web UI (`basic_connect`) | ~116 KB | ~1 MB | 16 KB | ~226 KB |
| WiFi + Cellular failover (`failover_connect`) | 123 KB | 950 KB | 16 KB | 219 KB |
| Cellular only (`cellular_connect`) | 85 KB | 758 KB | 16 KB | 256 KB |

### Runtime Memory (allocated from heap at startup)

| Resource | Size | Location |
|----------|------|----------|
| Task stacks (coord + derp_tx + net_io + wg_mgr) | 42 KB | SRAM |
| H2 receive buffer | 512 KB (configurable) | PSRAM |
| JSON parse buffer | 512 KB (configurable) | PSRAM |
| NVS peer cache (64 peers) | ~6 KB | PSRAM |
| HTTP config server | ~7 KB | SRAM (ifdef-gated) |
| Per WG peer | ~200 bytes | SRAM |

Boards without PSRAM can reduce the H2/JSON buffers to 64KB via menuconfig (sufficient for ~30 peers).

## Examples

| Example | Description | Hardware |
|---------|-------------|----------|
| `basic_connect` | WiFi → Tailscale → UDP echo + authenticated web config + SoftAP provisioning | Any ESP32 with WiFi |
| `cellular_connect` | 4G cellular → Tailscale → bidirectional UDP | XIAO + Waveshare SIM7600 |
| `cellular_heartbeat` | Periodic heartbeat over 4G cellular | XIAO + Waveshare SIM7600 |
| `failover_connect` | WiFi primary + cellular fallback | XIAO + Waveshare SIM7600 |

## Architecture

MicroLink v2 uses a fully async, task-based architecture. All protocol operations run concurrently in dedicated FreeRTOS tasks with queue-based IPC — no polling loop needed.

```
WiFi/Cellular (lwIP netif)
        │
   ┌────▼─────┐   ┌──────────┐   ┌───────────┐
   │ net_io   │──▶│ wg_mgr   │──▶│ coord     │──▶ Control plane (ts2021)
   │ (UDP)    │   │ (WG)     │   │ (HTTP/2)  │
   └────▲─────┘   └──────────┘   └───────────┘
        │
   ┌────┴─────┐
   │ derp_tx  │──▶ DERP relay
   └──────────┘
```

### Task Stack Sizes

| Task | Stack (bytes) | Core | Priority |
|------|---------------|------|----------|
| `net_io` | 8192 | 1 | 5 |
| `wg_mgr` | 8192 | 1 | 4 |
| `coord` | 8192 | 0 | 4 |
| `derp_tx` | 4096 | 1 | 3 |
| HTTP config server | 6144 | (httpd) | default |

## HTTP Config Server

Enable `CONFIG_ML_ENABLE_CONFIG_HTTPD=y` (on by default in the examples) to get a web UI accessible at `http://<vpn-ip>/` from any device on your tailnet.

### Authentication

Every endpoint (including `/` and all `/api/*`) requires **HTTP Basic authentication**:

- Username: `admin`
- Password: the admin password, stored in NVS and seeded at first boot from `CONFIG_ML_CONFIG_PASSWORD` (default `microlink` — **change it**).
- Change it from the web UI (**Security → Change Password**), or `POST /api/password`. Takes effect immediately; the browser re-prompts on the next request.
- **Fail-closed**: if the stored password is empty, every request returns `401`. Recover by re-flashing or erasing NVS.

> Note: HTTP Basic credentials travel as Base64 (not encrypted). Over the tailnet the traffic is protected by WireGuard; over plain LAN it is not. This is a documented limitation — use HTTPS if end-to-end encryption is required.

### System Monitor

Real-time ESP32 temperature, WiFi RSSI (or cellular indicator), uptime, heap/PSRAM usage, DERP region, peer count, per-task stack watermarks. Auto-refreshes every 3 seconds. The WiFi section shows the **current connection status** (SSID, RSSI, channel).

### WiFi Management

- Multi-SSID list with priority ordering (drag to reorder, restart to apply).
- **Scan** — scan for nearby APs (SSID, RSSI, security) directly from the page and pick one to connect.
- Connection status display (connected SSID / RSSI / channel).

### Subnet Router (advertise_routes)

- Configure advertised LAN subnets (comma-separated CIDRs, e.g. `10.0.0.0/8, 192.168.0.0/16`) in **Device Settings → Advertised Routes**.
- Persisted to NVS (overrides the Kconfig default) and applied on restart — no reflash.
- Requires `CONFIG_LWIP_IP_FORWARD` + `CONFIG_LWIP_IPV4_NAPT`, and route approval in the Tailscale admin console.

### Peer Allowlist

Manage which peers receive DISCO probes. Changes take effect immediately (no restart). On large tailnets (200+ devices), this limits DISCO probing to only the peers that matter.

### REST API

All functionality is available via JSON endpoints (all require Basic auth):

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/settings` | GET/POST | Read/write device settings (incl. `advertise_routes`) |
| `/api/peers` | GET | List all tailnet peers |
| `/api/peers/allowed` | GET/POST/DELETE | Manage DISCO probe allowlist |
| `/api/monitor` | GET | System health (temp, heap, RSSI, uptime, tasks) |
| `/api/status` | GET | Connection state and VPN IP |
| `/api/wifi` | GET/POST | Read/replace the WiFi network list |
| `/api/wifi/scan` | GET | Scan for nearby APs |
| `/api/password` | POST | Change the admin password |
| `/api/restart` | POST | Restart the device |

Example — read settings:

```bash
curl -u admin:yourpassword http://<vpn-ip>/api/settings
```

## API Reference

### Initialization

```c
microlink_t *microlink_init(const microlink_config_t *config);
esp_err_t microlink_start(microlink_t *ml);
```

### Connection Status

```c
bool microlink_is_connected(microlink_t *ml);
microlink_state_t microlink_get_state(microlink_t *ml);
uint32_t microlink_get_vpn_ip(microlink_t *ml);
int microlink_get_peer_count(microlink_t *ml);
```

### UDP Communication

```c
microlink_udp_socket_t *microlink_udp_create(microlink_t *ml, uint16_t port);
esp_err_t microlink_udp_send(microlink_udp_socket_t *sock, uint32_t dest_ip, uint16_t dest_port, const uint8_t *data, size_t len);
void microlink_udp_set_rx_callback(microlink_udp_socket_t *sock, microlink_udp_rx_cb_t cb, void *user_data);
```

### TCP Communication

```c
microlink_tcp_socket_t *microlink_tcp_connect(microlink_t *ml, uint32_t dest_ip, uint16_t dest_port);
esp_err_t microlink_tcp_send(microlink_tcp_socket_t *sock, const uint8_t *data, size_t len);
```

### Peer Information

```c
esp_err_t microlink_get_peer_info(microlink_t *ml, int index, microlink_peer_info_t *info);
```

### Callbacks

```c
void microlink_set_state_callback(microlink_t *ml, microlink_state_cb_t cb, void *data);
void microlink_set_peer_callback(microlink_t *ml, microlink_peer_cb_t cb, void *data);
```

### Cleanup / Reset

```c
esp_err_t microlink_factory_reset(void);
```

## Configuration

All settings via `idf.py menuconfig` → MicroLink V2 Configuration.

### Runtime Configuration

| Option | Default | Description |
|--------|---------|-------------|
| `auth_key` | Required | Tailscale auth key (reusable + ephemeral recommended) |
| `device_name` | Auto-generated | Device hostname (`esp32-XXYYZZ` from MAC, or `prefix-IMEI` for cellular) |
| `advertise_routes` | Empty | Subnet router advertised CIDRs (also editable in web UI) |
| `enable_derp` | `true` | Enable DERP relay |
| `enable_disco` | `true` | Enable DISCO path discovery |
| `enable_stun` | `true` | Enable STUN NAT discovery |
| `max_peers` | `16` | Maximum simultaneous active WireGuard tunnels |
| `priority_peer_ip` | `0` | VPN IP of priority peer (guaranteed WG slot) |
| `disco_heartbeat_ms` | `3000` | DISCO keepalive interval |
| `stun_interval_ms` | `23000` | STUN re-probe interval |
| `ctrl_watchdog_ms` | `120000` | Control plane watchdog timeout |
| `wifi_tx_power_dbm` | `0` (default) | WiFi TX power in dBm |

### Kconfig Options

- **Credentials** — `ML_WIFI_SSID`, `ML_WIFI_PASSWORD`, `ML_TAILSCALE_AUTH_KEY`, `ML_DEVICE_NAME` (git-ignored `sdkconfig`)
- **HTTP Config Server** — `ML_ENABLE_CONFIG_HTTPD`, `ML_CONFIG_MAX_ALLOWED_PEERS`, `ML_CONFIG_PASSWORD` (initial web admin password)
- **Subnet router** — `ML_ADVERTISE_ROUTES` (Kconfig default, overridable from web UI)
- **Network switching** — `ML_ENABLE_NET_SWITCH` and related timeouts
- **High throughput** — `ML_ZERO_COPY_WG`
- **Buffers** — `ML_H2_BUFFER_SIZE_KB`, `ML_JSON_BUFFER_SIZE_KB`, `ML_MAX_PEERS`

### DERP Relay

DERP relays are discovered dynamically from the control plane (`derpmap`). Fallback to manual region selection is supported for networks without public internet access to the default DERP map.

### Custom Coordination Server (Headscale / Ionscale)

Set `ctrl_host` (web UI → Device Settings, or `ML_CONFIG_CTRL_HOST`) to point at your own coordination server. TLS verification is configurable for self-signed certificates.

## Troubleshooting

### Device not appearing in tailnet
- Check the auth key is valid and reusable/ephemeral.
- Check `tailscale ping <hostname>` from another device.
- If the device shows as offline in the admin console after an NVS erase, its keys regenerated — it now appears as a **new node** with a new VPN IP.

### `tailscale ping` times out
- Verify WiFi/cellular connectivity; check DERP region status in the monitor.
- Large tailnets: enable the peer allowlist to limit DISCO probing.

### App partition too small
- Ensure `CONFIG_PARTITION_TABLE_SINGLE_APP_LARGE=y` and a clean build: `rm -rf build sdkconfig && idf.py build`.

### Cannot reach the config page in a new environment
- The device should have opened the `MicroLink-Config` SoftAP — connect to it and browse to `http://192.168.4.1`.
- If it is not broadcasting, confirm the device is powered and not already connected to a network.

### Forgot the admin password
- The password is stored in NVS. Recover by erasing NVS (re-flashes the Kconfig default) or re-flashing the firmware.

## License

MIT License — see [LICENSE](LICENSE)

## Acknowledgments

- [Tailscale](https://tailscale.com/) for the protocol specification
- [Headscale](https://github.com/juanfont/headscale) for open-source coordination server insights
- [WireGuard](https://www.wireguard.com/) for the cryptographic foundation
- [lwIP](https://savannah.nongnu.org/projects/lwip/) for the TCP/IP stack
- [wireguard-lwip](https://github.com/smartalock/wireguard-lwip) for WireGuard-lwIP integration

## Disclaimer

This is an independent implementation created for educational and interoperability purposes. It is not affiliated with or endorsed by Tailscale Inc. Use at your own risk.
