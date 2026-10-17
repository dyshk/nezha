# 哪吒面板 Nginx 反代配置（CDN 资源本地化 + gRPC / WebSocket）

## 1. 下载本地化 CSS 资源

```bash
sudo mkdir -p /var/www/cdn
sudo curl -sSL -o /var/www/cdn/flag-icons.min.css https://fastly.jsdelivr.net/gh/lipis/flag-icons@7.0.0/css/flag-icons.min.css
sudo curl -sSL -o /var/www/cdn/font-logos.css https://fastly.jsdelivr.net/npm/font-logos@1/assets/font-logos.css
```

## 2. Nginx 配置

```nginx
upstream dashboard {
    server 127.0.0.1:8008;
    keepalive 512;
}

server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name 你的服务器ip;

    underscores_in_headers on;
    server_tokens off;

    # ---- 压缩：Nginx -> 浏览器 这一段的体积优化 ----
    gzip on;
    gzip_vary on;
    gzip_comp_level 5;
    gzip_min_length 1024;
    gzip_proxied any;
    gzip_types text/plain text/css application/javascript application/json image/svg+xml;

    # ---- gRPC：Agent 连接（Agent 地址改到 80/443 后才会走这里） ----
    location ^~ /proto.NezhaService/ {
        grpc_set_header Host $host;
        grpc_set_header nz-realip $remote_addr;
        grpc_read_timeout 600s;
        grpc_send_timeout 600s;
        grpc_socket_keepalive on;
        client_max_body_size 10m;
        grpc_buffer_size 4m;
        grpc_pass grpc://dashboard;
    }

    # ---- 静态资源：长效缓存，第二次访问直接命中本地 ----
    location ^~ /assets/ {
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header Accept-Encoding "";
        proxy_pass http://127.0.0.1:8008;
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # ---- WebSocket：前台实时数据 ----
    location ~* ^/api/v1/ws/(server|terminal|file)(.*)$ {
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header nz-realip $remote_addr;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
        proxy_pass http://127.0.0.1:8008;
    }

    # ---- 本地化 CSS（需先下载到 /var/www/cdn/，零回源） ----
    location = /cdn/flag-icons.min.css {
        alias /var/www/cdn/flag-icons.min.css;
        default_type text/css;
        expires 30d;
        access_log off;
    }

    location = /cdn/font-logos.css {
        alias /var/www/cdn/font-logos.css;
        default_type text/css;
        expires 30d;
        access_log off;
    }

    # ---- Web 主入口 ----
    location / {
        proxy_http_version 1.1;
        proxy_set_header Accept-Encoding "";
        proxy_set_header Host $host;
        proxy_set_header nz-realip $remote_addr;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
        proxy_buffer_size 128k;
        proxy_buffers 4 256k;
        proxy_busy_buffers_size 256k;
        proxy_max_temp_file_size 0;
        proxy_pass http://127.0.0.1:8008;

        sub_filter_once off;
        sub_filter_types text/html;
        sub_filter 'https://fastly.jsdelivr.net/gh/lipis/flag-icons@7.0.0/css/flag-icons.min.css' '/cdn/flag-icons.min.css';
        sub_filter 'https://fastly.jsdelivr.net/npm/font-logos@1/assets/font-logos.css' '/cdn/font-logos.css';
    }
}
```

## 3. 配置要点说明

| 模块 | 作用 |
|------|------|
| `upstream dashboard` | 后端哪吒面板 `127.0.0.1:8008`，开启 `keepalive 512` 长连接复用 |
| gzip 段 | Nginx → 浏览器的响应压缩，降低 HTML/CSS/JS/JSON/SVG 传输体积 |
| `location ^~ /proto.NezhaService/` | gRPC 流量入口，Agent 需将地址改到 80/443；通过 `nz-realip` 头传递真实 IP |
| `location ^~ /assets/` | 静态资源 30 天长效缓存（`public, immutable`），关闭访问日志 |
| `location ~* ^/api/v1/ws/...` | WebSocket（server/terminal/file 实时数据），1 小时超时 |
| `location = /cdn/*.css` | 本地托管的 flag-icons 与 font-logos 样式，30 天缓存、零回源 |
| `location /` 主入口 | 通用反代 + `sub_filter` 将页面中 jsDelivr CDN 链接改写为本地 `/cdn/` 路径 |

> 注意：`server_name 32.188.73.145` 需替换为你自己的服务器 IP 或域名；使用 HTTPS 时还需补充 443 监听与 SSL 证书配置。

## 4. 钉钉每日服务器状态推送脚本

<img width="1065" height="843" alt="image" src="https://github.com/user-attachments/assets/c8bbf931-674d-411a-8eaf-a77490999184" />
<img width="1058" height="916" alt="image" src="https://github.com/user-attachments/assets/be0be53c-b27b-4018-9e7a-e82e9ded2094" />
<img width="1088" height="738" alt="image" src="https://github.com/user-attachments/assets/4c375055-6d40-43dc-95bb-916aca01975d" />

文件位置：`/opt/nezha/daily_report.py`（配合哪吒面板「任务 → 计划任务」使用，由 Agent 代跑，不需要安装任何常驻服务）。

用法：

```bash
/usr/bin/python3 /opt/nezha/daily_report.py
```

特点：

- 只用 Python 标准库，无需 pip 安装任何依赖
- 时间强制按北京时间计算（`gmtime + 8h`），不受服务器时区影响
- Markdown 卡片式排版，手机上也好读
- 拉取数据失败时以非 0 退出码结束，哪吒会判定为「任务失败」并通过通知组告警

> 只需修改脚本内「配置区」的 `TOKEN` / `WEBHOOK` / `SECRET` 等项即可使用。

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""哪吒监控 -> 钉钉  每日服务器状态

配合哪吒面板「任务 → 计划任务」使用，由 Agent 代跑，不需要装任何服务。

用法：
    /usr/bin/python3 /opt/nezha/daily_report.py

特点：
  - 只用 Python 标准库，不用 pip 装东西
  - 时间强制按北京时间算，不受服务器时区影响
  - Markdown 排版 + 文本进度条，手机上也好读
  - 出问题会以非 0 退出码结束，哪吒会把它当「任务失败」并通过通知组告警

只需要改下面「配置区」的内容。
"""

import json
import math
import time
import base64
import hashlib
import hmac
import sys
import re
import datetime
import urllib.parse
import urllib.request

# ============================ 配置区（只改这里） ============================
TOKEN   = "你的哪吒监控API：nzp_V14gIMj8MIY04IB……"
WEBHOOK = "你的钉钉API：https://oapi.dingtalk.com/robot/send?access_token=290a8023c1919……"
SECRET  = "你的钉钉密钥：SEC3eb54c102cbcdc42c24b9bdd9a31b3efa7cba55……"

STYLE   = "markdown"            # markdown（推荐） / text（纯文本）
SHOW_SPEED = True               # True=显示实时网速那行，False=不显示
SPEED_FULL_MBPS = 10            # 网速进度条"满格"对应多少 MB/s（按你的带宽改）
API = "http://127.0.0.1:8008/api/v1/server"
OFFLINE_SEC = 120               # 超过多少秒没上报算离线
KEYWORD = "哪吒探针"             # 关键词模式下这就是关键词，别改
BAR_WIDTH = 10                  # 进度条格子数
# ===========================================================================


def beijing_now(fmt="%Y-%m-%d %H:%M"):
    """北京时间。用 gmtime + 8 小时，与服务器时区无关"""
    return time.strftime(fmt, time.gmtime(time.time() + 8 * 3600))


def g(d, *keys):
    """兼容取值：v2 API 返回 snake_case（name/state/load_5），兼容旧版 PascalCase"""
    for k in keys:
        v = (d or {}).get(k)
        if v is not None:
            return v
    return None


def human_bytes(n):
    n = float(n or 0)
    for unit in ("B", "KB", "MB", "GB", "TB"):
        if n < 1024:
            return "%.1f%s" % (n, unit)
        n /= 1024
    return "%.1fTB" % n


def bar(pct):
    """文本进度条：▓▓▓░░░░░░░"""
    pct = min(max(float(pct or 0), 0.0), 100.0)
    n = pct / 100.0 * BAR_WIDTH
    filled = math.floor(n + 0.5)          # 四舍五入
    if filled == 0 and n > 0:             # 有负载就至少显示一格
        filled = 1
    filled = min(max(filled, 0), BAR_WIDTH)
    return "▓" * filled + "░" * (BAR_WIDTH - filled)


def speed_bar(bps):
    """网速进度条：按 SPEED_FULL_MBPS 作为满格"""
    mb = float(bps or 0) / 1024.0 / 1024.0
    pct = min(mb / float(SPEED_FULL_MBPS) * 100.0, 100.0)
    return bar(pct)


def level_icon(pct):
    pct = float(pct or 0)
    if pct >= 90:
        return "🔴"
    if pct >= 70:
        return "🟠"
    return "🟢"


def _parse_time(text):
    """解析哪吒的时间戳（可能是纳秒精度）"""
    text = (text or "").replace("Z", "+00:00")
    text = re.sub(r"\.(\d{6})\d*", r".\1", text)
    return datetime.datetime.fromisoformat(text)


def is_offline(s):
    last = g(s, "last_active", "LastActive") or ""
    if not last:
        return True
    try:
        delta = datetime.datetime.now(datetime.timezone.utc) - _parse_time(last)
        return delta.total_seconds() > OFFLINE_SEC
    except Exception:
        return False


def last_active_cn(s):
    last = g(s, "last_active", "LastActive") or ""
    try:
        cn = _parse_time(last).astimezone(datetime.timezone(datetime.timedelta(hours=8)))
        return cn.strftime("%m-%d %H:%M")
    except Exception:
        return "未知"


def uptime_cn(sec):
    sec = int(sec or 0)
    d, h, m = sec // 86400, sec % 86400 // 3600, sec % 3600 // 60
    if d:
        return "%d天%d小时" % (d, h)
    if h:
        return "%d小时%d分" % (h, m)
    return "%d分钟" % m


def snapshot(s):
    """把原始返回整理成统一字典"""
    host = g(s, "host", "Host") or {}
    state = g(s, "state", "State") or {}
    mt = float(g(host, "mem_total", "MemTotal") or 0) or 1
    dt = float(g(host, "disk_total", "DiskTotal") or 0) or 1
    return {
        "name": g(s, "name", "Name") or ("server-" + str(g(s, "id", "ID"))),
        "platform": " ".join(x for x in (
            g(host, "platform", "Platform"),
            g(host, "platform_version", "PlatformVersion")) if x) or "-",
        "cpu": float(state.get("cpu") or state.get("CPU") or 0),
        "mem_pct": float(state.get("mem_used") or state.get("MemUsed") or 0) / mt * 100,
        "mem_used": float(state.get("mem_used") or state.get("MemUsed") or 0),
        "mem_total": mt,
        "disk_pct": float(state.get("disk_used") or state.get("DiskUsed") or 0) / dt * 100,
        "disk_used": float(state.get("disk_used") or state.get("DiskUsed") or 0),
        "disk_total": dt,
        "load1": float(state.get("load_1") or state.get("Load1") or 0),
        "load5": float(state.get("load_5") or state.get("Load5") or 0),
        "load15": float(state.get("load_15") or state.get("Load15") or 0),
        "up": float(state.get("net_out_speed") or state.get("NetOutSpeed") or 0),
        "down": float(state.get("net_in_speed") or state.get("NetInSpeed") or 0),
        "tx": float(state.get("net_out_transfer") or state.get("NetOutTransfer") or 0),
        "rx": float(state.get("net_in_transfer") or state.get("NetInTransfer") or 0),
        "uptime": int(state.get("uptime") or state.get("Uptime") or 0),
        "process": int(state.get("process_count") or state.get("ProcessCount") or 0),
        "tcp": int(state.get("tcp_conn_count") or state.get("TcpConnCount") or 0),
        "online": not is_offline(s),
        "last": last_active_cn(s),
    }


# ---- 卡片式双列排版（仿 Komari 风格） ----
#
# 钉钉 Markdown 的两个坑（实测）：
#   1. 单个 \n 不换行，必须用空行 \n\n；
#   2. ** 两侧的空格会被吞掉，所以两列之间必须用可见分隔符 ｜，
#      不能靠空格对齐。

SEP = "｜"        # 列分隔符（前后空格写在格式串里，避免被压缩成双空格）
BR = "\n\n"       # 钉钉唯一可靠的换行


def card(s):
    """一台服务器的小卡片"""
    icon = "🔴" if not s["online"] else level_icon(max(s["cpu"], s["mem_pct"], s["disk_pct"]))
    rows = ["%s **%s** ｜ %s" % (icon, s["name"], s["platform"])]
    if not s["online"]:
        rows.append("🚫 离线 ｜ 最后上报 %s" % s["last"])
        return BR.join(rows)

    rows.append("🕐 在线 " + uptime_cn(s["uptime"]))
    rows.append("🖥 CPU **%.1f%%** %s 负载 %.2f / %.2f / %.2f" % (
        s["cpu"], SEP, s["load1"], s["load5"], s["load15"]))
    rows.append("💾 内存 **%.1f%%** %s %s / %s" % (
        s["mem_pct"], SEP,
        human_bytes(s["mem_used"]), human_bytes(s["mem_total"])))
    rows.append("🔀 流量 **%s** %s 累计 ↑%s ↓%s" % (
        human_bytes(s["tx"] + s["rx"]), SEP,
        human_bytes(s["tx"]), human_bytes(s["rx"])))
    rows.append("💽 硬盘 **%.1f%%** %s %s / %s" % (
        s["disk_pct"], SEP,
        human_bytes(s["disk_used"]), human_bytes(s["disk_total"])))
    if SHOW_SPEED:
        rows.append("🌐 网速 ↑ **%s/s** ｜ ↓ **%s/s**" % (
            human_bytes(s["up"]), human_bytes(s["down"])))
    else:
        rows.append("进程 %d ｜ 连接 %d" % (s["process"], s["tcp"]))
    return BR.join(rows)


def build_markdown(servers):
    now = beijing_now()
    snaps = [snapshot(s) for s in servers]
    online = [x for x in snaps if x["online"]]
    off = [x for x in snaps if not x["online"]]

    status = "🟢 全部在线" if not off else "🔴 %d 台离线" % len(off)
    parts = ["### 🖥 " + KEYWORD + " · 服务器日报",
             "📅 " + now + " · " + status]

    for s in online:
        parts.append(card(s))
    for s in off:
        parts.append(card(s))

    if len(snaps) > 1:
        avg = sum(x["mem_pct"] for x in online) / max(len(online), 1)
        parts.append("— 共 %d 台 · 在线 %d · 平均内存 %.0f%%" % (
            len(snaps), len(online), avg))
    return BR.join(parts)


def build_text(servers):
    """纯文本版，兼容性最好"""
    now = beijing_now()
    snaps = [snapshot(s) for s in servers]
    off = [x for x in snaps if not x["online"]]
    lines = [KEYWORD + " · 每日服务器状态 · " + now, ""]
    for s in snaps:
        if not s["online"]:
            lines.append("○ %s  离线（最后上报 %s）" % (s["name"], s["last"]))
            continue
        lines.append("● %s  CPU %.0f%% | 内存 %.0f%% | 磁盘 %.0f%%" % (
            s["name"], s["cpu"], s["mem_pct"], s["disk_pct"]))
        lines.append("   负载 %.2f/%.2f | ↑%s/s ↓%s/s | 运行 %s" % (
            s["load1"], s["load5"], human_bytes(s["up"]),
            human_bytes(s["down"]), uptime_cn(s["uptime"])))
    tail = ("，离线：" + "、".join(x["name"] for x in off)) if off else "，全部在线"
    lines.append("")
    lines.append("共 %d 台%s" % (len(snaps), tail))
    return "\n".join(lines)


def send(title, content, style="markdown"):
    url = WEBHOOK
    if SECRET:
        ts = str(round(time.time() * 1000))
        sign = urllib.parse.quote_plus(base64.b64encode(hmac.new(
            SECRET.encode(),
            ("%s\n%s" % (ts, SECRET)).encode(),
            hashlib.sha256,
        ).digest()))
        url += ("&" if "?" in WEBHOOK else "?") + "timestamp=%s&sign=%s" % (ts, sign)

    if style == "markdown":
        payload = {"msgtype": "markdown", "markdown": {"title": title, "text": content}}
    else:
        payload = {"msgtype": "text", "text": {"content": content}}

    body = json.dumps(payload, ensure_ascii=False).encode("utf-8")
    req = urllib.request.Request(
        url, data=body, headers={"Content-Type": "application/json; charset=utf-8"}
    )
    return urllib.request.urlopen(req, timeout=15).read().decode()


def main():
    try:
        req = urllib.request.Request(API, headers={"Authorization": "Bearer " + TOKEN})
        raw = json.load(urllib.request.urlopen(req, timeout=15))
        servers = (raw.get("data") if isinstance(raw, dict) else raw) or []
    except Exception as exc:
        err = "%s: %s" % (type(exc).__name__, exc)
        try:
            send("服务器日报失败",
                 KEYWORD + "：拉取哪吒数据失败\n" + err, style="text")
        except Exception:
            pass
        print("拉取哪吒数据失败：" + err, file=sys.stderr)
        return 1

    title = "服务器日报 · " + beijing_now("%m-%d %H:%M")
    content = build_markdown(servers) if STYLE == "markdown" else build_text(servers)
    print(send(title, content, style=STYLE))
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

### 4.1 配置区说明

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `TOKEN` | `nzp_...` | 哪吒面板 API Token，用于请求 `/api/v1/server` |
| `WEBHOOK` | `oapi.dingtalk.com/robot/send?access_token=...` | 钉钉自定义机器人 Webhook 地址 |
| `SECRET` | `SEC...` | 钉钉机器人加签密钥，用于 HMAC-SHA256 签名 |
| `STYLE` | `markdown` | `markdown`（推荐）或 `text`（纯文本，兼容性最好） |
| `SHOW_SPEED` | `True` | 是否显示实时网速那一行；关闭则显示进程数与连接数 |
| `SPEED_FULL_MBPS` | `10` | 网速进度条「满格」对应的 MB/s，按实际带宽调整 |
| `API` | `http://127.0.0.1:8008/api/v1/server` | 哪吒面板本机 API 地址 |
| `OFFLINE_SEC` | `120` | 超过多少秒未上报即判定为离线 |
| `KEYWORD` | `哪吒探针` | 钉钉机器人「关键词模式」下的关键词 |
| `BAR_WIDTH` | `10` | 进度条格子数 |

### 4.2 部署步骤

1. 将上述源码保存为 `/opt/nezha/daily_report.py`
2. 赋予执行权限：`chmod +x /opt/nezha/daily_report.py`
3. 手动试跑一次，确认钉钉能收到消息：`/usr/bin/python3 /opt/nezha/daily_report.py`
4. 在哪吒面板「任务 → 计划任务」中新建任务，命令填 `/usr/bin/python3 /opt/nezha/daily_report.py`，按需要设置每日触发时间

### 4.3 实现要点

- **字段兼容**：`g()` 函数同时兼容 v2 API 的 snake_case（`name` / `state` / `load_5`）与旧版 PascalCase（`Name` / `State` / `Load5`）
- **时间戳解析**：`_parse_time()` 将纳秒精度截断为微秒，兼容哪吒返回的高精度时间戳
- **钉钉换行**：单个 `\n` 在钉钉 Markdown 中不换行，必须使用空行 `\n\n`（脚本中记为 `BR`）
- **列分隔**：`**` 两侧的空格会被吞掉，因此两列之间使用可见分隔符 `｜`（`SEP`）而非空格对齐
- **离线判定**：`last_active` 为空或超过 `OFFLINE_SEC` 秒未上报即标记离线，卡片显示最后上报时间


## 5. 独立版：不依赖哪吒的日报脚本

文件位置：`/opt/nezha/standalone_report.py`。直接读本机 `/proc` 采集数据，**不需要哪吒面板、不需要 Agent、不需要 Token**，只用 Python 标准库。

用法：

```bash
/usr/bin/python3 /opt/nezha/standalone_report.py
```

> 说明：独立版只上报**本机**状态；若有多台机器，需每台都放一份并各自设定时任务。它可以与第 4 节的哪吒版共存（两个都要跑时请错开时间）。

### 5.1 配置区说明

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `WEBHOOK` | `oapi.dingtalk.com/robot/send?access_token=...` | 钉钉自定义机器人 Webhook 地址 |
| `SECRET` | `SEC...` | 钉钉机器人加签密钥，用于 HMAC-SHA256 签名 |
| `TITLE_PREFIX` | `哪吒探针` | 钉钉机器人「关键词模式」下的关键词 |
| `DISK_PATH` | `/` | 统计哪块盘的用量 |
| `SAMPLE_SEC` | `1.0` | CPU / 网速的采样间隔（秒） |
| `PROC` | `/proc` | 数据采集根目录，只读，一般不用改 |

### 5.2 源码

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""Linux 服务器状态 -> 钉钉（完全不依赖哪吒）

直接从本机 /proc 读数据，不需要哪吒面板、不需要 Agent、不需要 Token。
只用 Python 标准库，不用 pip 装任何东西。

用法：
    /usr/bin/python3 /opt/nezha/standalone_report.py

定时（任选一种）：
    crontab: 0 1 * * * /usr/bin/python3 /opt/nezha/standalone_report.py
    systemd: 见 README

只需要改下面「配置区」的内容。
"""

import os
import re
import time
import json
import base64
import hashlib
import hmac
import socket
import urllib.parse
import urllib.request

# ============================ 配置区（只改这里） ============================
WEBHOOK = "你的钉钉API：https://oapi.dingtalk.com/robot/send?access_token=290a8023c1919……"
SECRET  = "你的钉钉密钥：SEC3eb54c102cbcdc42c24b9bdd9a31b3efa7cba55……"

TITLE_PREFIX = "哪吒探针"        # 关键词模式下这就是关键词，别改
DISK_PATH = "/"                  # 统计哪块盘的用量
SAMPLE_SEC = 1.0                 # CPU / 网速的采样间隔（秒）
PROC = "/proc"                   # 只读，一般不用改
# ===========================================================================


def gmt8(fmt="%Y-%m-%d %H:%M"):
    """北京时间，不受系统时区影响"""
    return time.strftime(fmt, time.gmtime(time.time() + 8 * 3600))


def human_bytes(n):
    n = float(n or 0)
    for unit in ("B", "KB", "MB", "GB", "TB"):
        if n < 1024:
            return "%.1f%s" % (n, unit)
        n /= 1024
    return "%.1fTB" % n


def uptime_cn(sec):
    sec = int(sec or 0)
    d, h, m = sec // 86400, sec % 86400 // 3600, sec % 3600 // 60
    if d:
        return "%d天%d小时" % (d, h)
    if h:
        return "%d小时%d分" % (h, m)
    return "%d分钟" % m


# ---------------- 采集层（全部来自 /proc，纯标准库） ----------------

def read(path):
    try:
        with open(path, "r", encoding="utf-8", errors="ignore") as fh:
            return fh.read()
    except Exception:
        return ""


def cpu_snapshot():
    """返回 (total, idle) 累计 jiffies"""
    line = read(os.path.join(PROC, "stat")).splitlines()
    for ln in line:
        if ln.startswith("cpu "):
            parts = ln.split()
            # user nice system idle iowait irq softirq steal
            vals = [int(x) for x in parts[1:8]]
            # user nice system idle iowait irq softirq
            idle = vals[3] + (vals[4] if len(vals) > 4 else 0)   # idle + iowait
            return sum(vals), idle
    return 0, 0


def mem_info():
    info = {}
    for ln in read(os.path.join(PROC, "meminfo")).splitlines():
        m = re.match(r"^(\w+):\s+(\d+)", ln)
        if m:
            info[m.group(1)] = int(m.group(2)) * 1024   # kB -> B
    total = info.get("MemTotal", 1)
    avail = info.get("MemAvailable", info.get("MemFree", 0))
    used = max(total - avail, 0)
    return used, total


def disk_info(path=DISK_PATH):
    if not hasattr(os, "statvfs"):      # 非 Linux（如 Windows）没有这个接口
        return 0, 1
    try:
        st = os.statvfs(path)
        total = st.f_blocks * st.f_frsize
        free = st.f_bavail * st.f_frsize
        used = max(total - free, 0)
        return used, total
    except Exception:
        return 0, 1


def net_snapshot():
    """返回 (recv_bytes, sent_bytes)，跳过 lo"""
    rx = tx = 0
    for ln in read(os.path.join(PROC, "net", "dev")).splitlines()[2:]:
        if ":" not in ln:
            continue
        iface, data = ln.split(":", 1)
        if iface.strip() == "lo":
            continue
        cols = data.split()
        rx += int(cols[0])    # receive bytes
        tx += int(cols[8])    # transmit bytes
    return rx, tx


def system_uptime():
    txt = read(os.path.join(PROC, "uptime")).split()
    return float(txt[0]) if txt else 0


def load_avg():
    try:
        return os.getloadavg()
    except Exception:
        return (0.0, 0.0, 0.0)


def collect():
    """采样一次完整状态"""
    c1 = cpu_snapshot()
    n1 = net_snapshot()
    time.sleep(SAMPLE_SEC)
    c2 = cpu_snapshot()
    n2 = net_snapshot()

    dt_total = max(c2[0] - c1[0], 1)
    dt_idle = max(c2[1] - c1[1], 0)
    cpu = max(0.0, min(100.0, (1 - dt_idle / float(dt_total)) * 100))

    mem_used, mem_total = mem_info()
    disk_used, disk_total = disk_info()
    rx, tx = n2
    speed_in = max(n2[0] - n1[0], 0) / SAMPLE_SEC
    speed_out = max(n2[1] - n1[1], 0) / SAMPLE_SEC
    l1, l5, l15 = load_avg()

    return {
        "host": socket.gethostname(),
        "cpu": cpu,
        "load": (l1, l5, l15),
        "mem_used": mem_used, "mem_total": mem_total,
        "mem_pct": mem_used / float(mem_total or 1) * 100,
        "disk_used": disk_used, "disk_total": disk_total,
        "disk_pct": disk_used / float(disk_total or 1) * 100,
        "rx": rx, "tx": tx,
        "up": speed_out, "down": speed_in,
        "uptime": system_uptime(),
    }


# ---------------- 展示层（与哪吒版同款卡片） ----------------

SEP = "｜"
BR = "\n\n"


def build_markdown(s):
    icon = "🔴" if max(s["cpu"], s["mem_pct"], s["disk_pct"]) >= 90 else (
        "🟠" if max(s["cpu"], s["mem_pct"], s["disk_pct"]) >= 70 else "🟢")
    rows = [
        "### 🖥 %s · 服务器日报" % TITLE_PREFIX,
        "📅 %s · %s 运行中" % (gmt8(), icon),
        "%s **%s** ｜ 🐧 Linux" % (icon, s["host"]),
        "🕐 已运行 " + uptime_cn(s["uptime"]),
        "🖥 CPU **%.1f%%** ｜ 负载 %.2f / %.2f / %.2f" % (
            s["cpu"], s["load"][0], s["load"][1], s["load"][2]),
        "💾 内存 **%.1f%%** ｜ %s / %s" % (
            s["mem_pct"], human_bytes(s["mem_used"]), human_bytes(s["mem_total"])),
        "🔀 流量 **%s** ｜ 累计 ↑%s ↓%s" % (
            human_bytes(s["tx"] + s["rx"]), human_bytes(s["tx"]), human_bytes(s["rx"])),
        "💽 硬盘 **%.1f%%** ｜ %s / %s" % (
            s["disk_pct"], human_bytes(s["disk_used"]), human_bytes(s["disk_total"])),
        "🌐 网速 ↑ **%s/s** ｜ ↓ **%s/s**" % (
            human_bytes(s["up"]), human_bytes(s["down"])),
    ]
    return BR.join(rows)


def send(title, content):
    url = WEBHOOK
    if SECRET:
        ts = str(round(time.time() * 1000))
        sign = urllib.parse.quote_plus(base64.b64encode(hmac.new(
            SECRET.encode(), ("%s\n%s" % (ts, SECRET)).encode(), hashlib.sha256).digest()))
        url += ("&" if "?" in WEBHOOK else "?") + "timestamp=%s&sign=%s" % (ts, sign)
    body = json.dumps(
        {"msgtype": "markdown", "markdown": {"title": title, "text": content}},
        ensure_ascii=False).encode("utf-8")
    req = urllib.request.Request(
        url, data=body, headers={"Content-Type": "application/json; charset=utf-8"})
    return urllib.request.urlopen(req, timeout=15).read().decode()


def main():
    s = collect()
    title = "服务器日报 · " + gmt8("%m-%d %H:%M")
    content = build_markdown(s)
    print(send(title, content))
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

### 5.3 部署与定时任务（crontab）

部署时依次执行：①先手动跑一次确认脚本没问题 → ②确认 python3 路径 → ③加定时任务。

```bash
# ① 先手动跑一次，确认脚本本身没问题
/usr/bin/python3 /opt/nezha/standalone_report.py
# ② 确认 python3 路径对（输出应该有 /usr/bin/python3）
ls -l /usr/bin/python3
# ③ 加定时任务
(crontab -l 2>/dev/null; echo "0 1 * * * /usr/bin/python3 /opt/nezha/standalone_report.py >> /var/log/standalone_report.log 2>&1") | crontab -
```

> `0 1 * * *` 是**服务器本地时间** 01:00；若服务器为 UTC，则对应北京时间 09:00。日志写入 `/var/log/standalone_report.log`。

删除定时任务（只删这一条，保留其他任务）：

```bash
crontab -l 2>/dev/null | grep -v standalone_report | crontab -
```

确认删干净了：

```bash
crontab -l          # 应该看不到 standalone_report 了
```

### 5.4 两个版本怎么选

| 对比项 | 第 4 节哪吒版 `daily_report.py` | 第 5 节独立版 `standalone_report.py` |
|--------|----------------------------------|----------------------------------------|
| 数据来源 | 哪吒面板 `/api/v1/server` REST API | 本机 `/proc`（stat / meminfo / net/dev / uptime） |
| 依赖 | 需面板 + Agent + Token | 无需任何外部组件，仅需 Python 3 标准库 |
| 上报范围 | 一次推送**所有**被监控机器，含离线判定 | 只上报**本机** |
| 离线告警 | 支持（`last_active` 超过 `OFFLINE_SEC` 判定） | 不支持（脚本在机器上跑，机器挂了也就发不出） |
| 适用场景 | 已部署哪吒、需要统一汇总多台机器 | 临时机器、不想装 Agent，或作为面板挂掉时的备份通道 |

实现要点：

- **CPU 计算**：采样间隔前后两次读取 `/proc/stat`，用 `(1 - Δidle / Δtotal) * 100` 得出；idle 计入 `idle + iowait`
- **网速**：两次读取 `/proc/net/dev` 求差并除以 `SAMPLE_SEC`，自动跳过 `lo` 回环网卡
- **内存**：`MemTotal - MemAvailable`（无 `MemAvailable` 时退化为 `MemFree`），与 `free` 命令口径一致
- **磁盘**：`os.statvfs()` 计算，非 Linux 环境（如 Windows）无此接口，已用 `hasattr` 保护并返回 0/N/A 值
