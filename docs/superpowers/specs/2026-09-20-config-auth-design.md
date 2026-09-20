# MicroLink v2 Web 配置页鉴权 设计

日期: 2026-09-20
状态: 已批准，进入实现

## 目标

给现有 HTTP 配置服务器（`ml_config_httpd.c` + `ml_config_html.h`）加上用户鉴权：
任何访问 ESP32 配置页/API 的人必须先登录才能查看和管理。
核心诉求：管理界面需要登录鉴权、页面保持极简、严格控制 APP 体积。

## 现状（探索结论）

项目已有一个完整 Web 配置服务器（`CONFIG_ML_ENABLE_CONFIG_HTTPD=y`）：
- WiFi 多 SSID 管理已具备（增删、优先级排序，存 NVS `wifi_list`，`GET/POST /api/wifi`）
- 完整仪表盘页面（状态监控、对端白名单、设备设置）
- **完全无鉴权** —— 任何能访问 ESP32 端口 80 的人都能改 WiFi 密码、Tailscale auth key

因此本需求真正新增的是：**① 用户鉴权（登录才能管理）② 密码管理**。
页面本身保持现有功能不变（用户已确认「给现有页面加鉴权」方案）。

## 已确认决策

| 决策项 | 选择 |
|---|---|
| 页面处理 | 给现有完整配置页加鉴权（保留仪表盘与全部功能） |
| 鉴权方式 | HTTP Basic Auth（浏览器原生登录框） |
| 密码管理 | NVS 存储（settings blob v4）+ 可改密码（Kconfig 种子初始值） |
| 鉴权范围 | 全部接口都要登录（含 `/`、`/api/status`、`/api/monitor` 等只读） |
| 监听接口 | 所有接口（保持现状，HTTP 明文；tailnet 内 WireGuard 加密，LAN 内明文） |
| 实现方案 | 方案 A：每个 handler 顶部统一鉴权门（宏 + helper） |

## 架构

```
浏览器 ──GET /──> ESP32 httpd :80
   │ 无 Authorization 头 → 401 + WWW-Authenticate: Basic realm="MicroLink"
   │                       └→ 浏览器弹原生登录框
   │ 带 Authorization: Basic base64("admin:<password>")
   │   └→ ml_config_auth_ok() 校验
   │        ├─ 通过 → 正常 handler 处理（12 个现有接口全部复用）
   │        └─ 失败 → 401
   └─ POST /api/password（登录态下改密）→ settings blob v4 持久化
```

要点：
- 浏览器 Basic 登录后，对同源所有请求（含现有 JS `fetch('/api/...')`）自动携带
  Authorization 头 → **现有页面 JS 完全不用改**，只有 Security 区是新增
- 密码仅存 NVS + Kconfig 种子，永不在 API 回读（不回显）

## 改动清单

### 1. 鉴权核心 (`src/ml_config_httpd.c`)

```c
/* 解析并校验 Authorization: Basic base64("admin:<password>") */
static bool ml_config_auth_ok(httpd_req_t *req, const char *stored_pass);
/* 发送 401 + WWW-Authenticate 并返回 ESP_FAIL */
static esp_err_t ml_config_auth_fail(httpd_req_t *req);

#define AUTH_GATE(req) do { \
    if (!ml_config_auth_ok(req, ctx->settings.admin_pass)) \
        return ml_config_auth_fail(req); \
} while (0)
```

`ml_config_auth_ok` 流程：
1. `httpd_req_get_hdr_value(req, "Authorization", ...)` 取头，校验 `Basic ` 前缀
2. `mbedtls_base64_decode` 解码（mbedtls 已链接，零新增依赖）
3. 拆分 `用户名:密码`，用户名固定 `admin`
4. 密码用常量时间比较 `mbedtls_ct_memcmp`（防时序侧信道）
5. 任一失败 → `ml_config_auth_fail`

接入：12 个现有 handler 顶部各加一行 `AUTH_GATE(req)`（含 `handler_root` 页面本身）。

### 2. 密码存储 (`include/ml_config_httpd.h`)

- `ML_CONFIG_SETTINGS_VERSION` 3 → 4
- 结构体末尾追加（packed 布局不变，v3 老 blob 读取后字段为 0）:
  ```c
  char admin_pass[64];   /* Web 管理密码（Basic auth） */
  ```
- `config_load_settings()`: `admin_pass` 为空时 `SEED_STR` 从 `CONFIG_ML_CONFIG_PASSWORD`
  种子，触发 v3→v4 迁移重存

### 3. Kconfig (`components/microlink/Kconfig`, HTTP Config Server 菜单)

```kconfig
config ML_CONFIG_PASSWORD
    string "Web admin password (HTTP Basic auth)"
    default "microlink"
    help
        Initial admin password for the config web UI. Stored in NVS and
        changeable at runtime via the web UI ("Change Password").
        WARNING: "microlink" is an insecure default — change it.
```

- 种子：NVS 有值用 NVS，否则用 Kconfig；改密后 NVS 覆盖 Kconfig
- **fail-closed**：NVS 与 Kconfig 都为空时所有请求 401（需重烧/擦 NVS 恢复），
  避免「无密码裸奔」

### 4. 改密接口 `POST /api/password`

```json
请求:  { "current": "旧密码", "new": "新密码" }
成功:  { "ok": true }
失败:  400 (新密码空/超长 63/与旧密码相同) 或 403 (current 不符)
```

- `current` 常量时间比对存储密码；`new` 长度 1~63、不得与当前相同
- 成功后写 `settings.admin_pass` → `config_save_settings(ctx)` → 立即生效
- 注册为第 13 个 URI handler

### 5. 前端 (`src/ml_config_html.h`)

Settings 区后加「Security」极简卡片（复用现有 CSS 类，零新增样式）：
- Current Password / New Password 两个输入框 + `Change Password` 按钮
- JS `changePassword()`: POST `/api/password`，成功提示后 `location.reload()`
  （触发浏览器重新弹登录框，旧凭据失效属预期行为）
- HTML 增量 ~0.3KB，JS 增量 ~0.3KB

## 体积影响

| 项 | 增量 |
|---|---|
| 鉴权代码（mbedtls 已链接，零新增依赖） | ~250B flash |
| settings blob +64B | NVS 无压力 |
| 页面 Security 区 | ~0.6KB flash |
| Kconfig 项 | 忽略 |

1500K 分区（SINGLE_APP_LARGE）充足。

## 安全说明

- HTTP 明文传输：Basic 凭据在网络中为 Base64（非加密）。tailnet 内经 WireGuard
  加密可接受；LAN 内为明文。如需端到端加密需上 HTTPS（mbedTLS 服务端证书，
  体积显著增加），本设计不做，记为已知限制。
- fail-closed：密码为空 → 全部 401，杜绝裸奔。
- 无暴力破解限速（个人 tailnet 风险低，YAGNI）。

## 验证方法

1. 构建 + 烧录 COM7（`idf_build.cmd`）
2. 串口日志：无崩溃、`admin password seeded (Kconfig/NVS)` 日志
3. 无凭据 `curl -i http://100.127.58.97/` → 401 + `WWW-Authenticate`
4. 错误密码 → 401；正确密码 `-u admin:microlink` → 200 JSON
5. `POST /api/password` 改密 → 旧密码 401，新密码 200
6. 浏览器：登录框 → 仪表盘 → 改密 → 重登录
7. 回归：`/api/wifi`、`/api/settings`、`/api/peers/allowed` 带凭据下功能不变
