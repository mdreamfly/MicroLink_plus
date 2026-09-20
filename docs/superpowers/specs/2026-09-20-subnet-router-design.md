# MicroLink v2 Subnet Router 设计

日期: 2026-09-20
状态: 已批准，进入实现

## 目标

让 ESP32 (MicroLink v2) 作为 Tailscale subnet router，通告 `10.39.0.0/16` 子网，
使 tailnet 中其他设备（PC/手机）能通过 ESP32 访问该网段的设备。

用户后续会将 ESP32 连接到 10.39.0.0/16 的 WiFi 环境，ESP32 的 WiFi 地址将是
10.39.x.x（与目标设备同一网段/二层）。

## 数据流

```
PC (100.64.0.2) ──WG隧道──> ESP32 WG netif (解密)
    → lwIP 转发 (IP_FORWARD): 目标 10.39.0.5 → WiFi netif
    → NAPT SNAT: 源改写为 ESP32 的 10.39.x.x WiFi 地址
    → 10.39.0.5 收到包，回包给 ESP32 (10.39.x.x)
    → ESP32 WiFi netif 收到，DNAT 查表还原目标 = 100.64.0.2
    → 路由到 WG netif → wireguardif_output 按目标 IP 匹配 peer → 加密回 PC
```

关键点（已验证）:
- wireguard-lwip 入站只校验**源 IP**（peer 的 tailnet IP /32），目标 IP 透传，
  无需修改 peer allowed_ip 逻辑
- ESP-IDF 6.1 lwIP 内置 OpenWrt 风格 NAPT (`ip4_napt.c`, `lwip_napt.h`),
  `ip_napt_enable_netif()` 可直接启用；IP_NAPT 依赖 IP_FORWARD
- NAPT 方向语义: `ip4.c` 转发路径调用 `ip_napt_forward()` 时要求 `inp->napt`
  (输入接口启用) —— 即在 **WG netif** 上启用 NAPT；回包 DNAT 走
  `ip4_input` → `ip_napt_recv()`

## 改动清单

### 1. 配置项
- `components/microlink/Kconfig`: 新增 `ML_ADVERTISE_ROUTES` (string, 逗号分隔 CIDR)
- `include/microlink.h`: `microlink_config_t` 新增 `advertise_routes` 字段
- 配置持久化 (NVS) 与 web UI 支持 (如有时间)

### 2. 控制平面通告 (`src/ml_coord.c`)
- `do_register()`: RegisterRequest 顶层加 `AdvertiseRoutes: ["10.39.0.0/16"]` 数组
- 两处 MapRequest 构建 (Stream=false 与 Stream=true): 同样加 `AdvertiseRoutes`
- Tailscale 控制协议 v131 原生支持该字段

### 3. 数据平面 (`src/ml_wg_mgr.c` + sdkconfig)
- `examples/basic_connect/sdkconfig.defaults`: 启用
  `CONFIG_LWIP_IP_FORWARD=y` + `CONFIG_LWIP_IPV4_NAPT=y`
- `wg_init_interface()`: netif up 后调用 `ip_napt_enable_netif(wg_netif, 1)`
  (需 `#include "lwip/ip4_napt.h"` 或 `lwip/lwip_napt.h`)

### 4. 后台操作（固件之外）
- Tailscale admin console 批准 `10.39.0.0/16` route
  (或创建 auth key 时带 `--advertise-routes=10.39.0.0/16` 预授权)
- 访问方设备执行 `tailscale up --accept-routes` (默认关闭)

## 验证方法
- 构建 + 烧录到 COM7
- 日志确认: RegisterRequest/MapRequest 含 AdvertiseRoutes; NAPT 启用日志
- 后台批准 route 后，从 PC `tailscale ping` 一台 10.39.x.x 设备
- ESP32 日志确认转发包计数、NAPT 表活动
