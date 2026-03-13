# EMQX Docker Compose 部署指南 / Deployment Guide

[中文](#中文) | [English](#english)

---

## 中文

### 簡介

本專案提供使用 Docker Compose 一鍵部署 **EMQX MQTT Broker** 的完整方案。EMQX 是全球領先的開源分散式 MQTT 訊息代理，專為 IoT、M2M 和行動應用設計，支援數百萬級並發連接。

本部署方案已在以下環境成功驗證：
- **EMQX 6.0.0** (`emqx/emqx:latest`)
- **Podman 4.9.3** + podman-compose 1.0.6
- **Ubuntu Linux** (也支援 Docker)

### 架構圖

```
┌──────────────────────────────────────────────┐
│              EMQX Broker v6.0.0              │
│  ┌──────────────────────────────────────────┐│
│  │  MQTT TCP:       1883                    ││
│  │  MQTT SSL/TLS:   8883                    ││
│  │  WebSocket:      8083                    ││
│  │  WebSocket SSL:  8084                    ││
│  │  Dashboard UI:   18083                   ││
│  └──────────────────────────────────────────┘│
│  Volumes:                                    │
│  ├── emqx_data (配置 + 運行數據)              │
│  └── emqx_log  (日誌)                        │
└──────────────────────────────────────────────┘
               ▲
               │ MQTT 連接
     ┌─────────┴─────────┐
     │   IoT 設備 / 感測器 │
     └───────────────────┘
```

### 系統需求

| 項目 | 最低要求 |
|------|---------|
| 容器引擎 | Docker 20.10+ 或 Podman 4.0+ |
| Compose 工具 | Docker Compose 2.0+ 或 podman-compose |
| 記憶體 | 512MB+ (建議 1GB+) |
| 端口 | 1883, 8883, 8083, 8084, 18083 |

### 專案檔案結構

```
Woow_eqmx_docker_compose_all/
├── docker-compose.yml   # Docker Compose 主要配置檔
├── .env.example         # 環境變數範例檔（複製為 .env 使用）
├── .gitignore           # Git 忽略規則（排除 .env 等敏感檔案）
├── README.md            # 本檔案 - 完整中英文部署說明
└── DEPLOY_SKILL.md      # AI 快速部署技能指引
```

### 快速開始

#### 1. 取得專案

```bash
git clone https://github.com/WOOWTECH/Woow_eqmx_docker_compose_all.git
cd Woow_eqmx_docker_compose_all
```

#### 2. 建立環境配置

```bash
cp .env.example .env
```

#### 3. 修改設定（建議）

編輯 `.env` 檔案，至少修改 Dashboard 密碼：

```bash
# 使用你喜好的編輯器
nano .env
# 或
vim .env
```

重要設定項：

| 變數 | 預設值 | 說明 |
|------|--------|------|
| `EMQX_VERSION` | latest | EMQX 映像版本 |
| `EMQX_DASHBOARD_USER` | admin | Dashboard 登入帳號 |
| `EMQX_DASHBOARD_PASSWORD` | public | Dashboard 登入密碼 (**請修改**) |
| `EMQX_DASHBOARD_PORT` | 18083 | Dashboard 網頁端口 |
| `MQTT_TCP_PORT` | 1883 | MQTT TCP 端口 |
| `MQTT_SSL_PORT` | 8883 | MQTT SSL 端口 |
| `MQTT_WS_PORT` | 8083 | MQTT WebSocket 端口 |
| `MQTT_WSS_PORT` | 8084 | MQTT WebSocket SSL 端口 |
| `EMQX_HOST` | 127.0.0.1 | 節點主機位址 |
| `EMQX_ALLOW_ANONYMOUS` | true | 是否允許匿名連接 |

#### 4. 啟動服務

**使用 Docker Compose:**
```bash
docker compose up -d
```

**使用 Podman:**
```bash
podman-compose up -d
```

#### 5. 驗證部署

等待約 30 秒讓 EMQX 完全啟動：

```bash
# 檢查容器狀態（STATUS 應顯示 healthy）
docker compose ps
# 或
podman-compose ps

# 檢查 EMQX 運行狀態
docker exec emqx emqx ctl status
# 或
podman exec emqx emqx ctl status
# 預期輸出: Node 'emqx@127.0.0.1' 6.0.0 is started

# 測試 Dashboard 是否可訪問
curl -s -o /dev/null -w "%{http_code}" http://localhost:18083
# 預期輸出: 200
```

#### 6. 登入 Dashboard

開啟瀏覽器訪問：**http://localhost:18083**

- 帳號：`admin`
- 密碼：`public`（或您在 `.env` 中設定的密碼）

### 端口說明

| 端口 | 協定 | 用途 | 預設開啟 |
|------|------|------|---------|
| 1883 | MQTT TCP | MQTT 標準連接 | 是 |
| 8883 | MQTT SSL | MQTT TLS 加密連接 | 是 |
| 8083 | WebSocket | MQTT over WebSocket | 是 |
| 8084 | WebSocket SSL | MQTT over WSS | 是 |
| 18083 | HTTP | Dashboard 管理介面 | 是 |

### 常用操作指令

```bash
# ===== 服務管理 =====
# 啟動
docker compose up -d

# 停止
docker compose down

# 重啟
docker compose restart

# 查看即時日誌
docker compose logs -f emqx

# ===== EMQX 管理 =====
# 進入 EMQX 容器
docker compose exec emqx sh

# 查看節點狀態
docker compose exec emqx emqx ctl status

# 列出已連接的客戶端
docker compose exec emqx emqx ctl clients list

# 列出所有主題
docker compose exec emqx emqx ctl topics list

# 查看叢集資訊
docker compose exec emqx emqx ctl cluster status
```

> **Podman 用戶**：將 `docker compose` 替換為 `podman-compose`，`docker exec` 替換為 `podman exec`。

### 測試 MQTT 連接

使用 mosquitto 客戶端工具進行測試：

```bash
# 安裝 mosquitto 客戶端
# Ubuntu/Debian
sudo apt install mosquitto-clients
# macOS
brew install mosquitto
# Alpine
apk add mosquitto-clients

# 在終端 1 訂閱主題
mosquitto_sub -h localhost -p 1883 -t "test/topic" -v

# 在終端 2 發布訊息
mosquitto_pub -h localhost -p 1883 -t "test/topic" -m "Hello EMQX!"

# 帶認證的連接（關閉匿名後）
mosquitto_sub -h localhost -p 1883 -t "test/#" -u "username" -P "password" -v
mosquitto_pub -h localhost -p 1883 -t "test/topic" -u "username" -P "password" -m "Hello!"
```

### 資料持久化

EMQX 數據存儲在 Docker/Podman volumes 中，容器移除後數據仍然保留：

- **emqx_data** - EMQX 配置檔案、規則引擎、認證數據
- **emqx_log** - 運行日誌

#### 備份

```bash
# 備份數據
docker run --rm -v emqx_data:/data -v $(pwd):/backup alpine tar czf /backup/emqx_data_backup.tar.gz /data

# 備份日誌
docker run --rm -v emqx_log:/data -v $(pwd):/backup alpine tar czf /backup/emqx_log_backup.tar.gz /data
```

#### 還原

```bash
# 還原數據
docker run --rm -v emqx_data:/data -v $(pwd):/backup alpine tar xzf /backup/emqx_data_backup.tar.gz -C /

# 還原日誌
docker run --rm -v emqx_log:/data -v $(pwd):/backup alpine tar xzf /backup/emqx_log_backup.tar.gz -C /
```

### 故障排除

| 問題 | 可能原因 | 解決方案 |
|------|---------|---------|
| 容器一直重啟 | 端口被佔用 | `ss -tlnp \| grep -E '1883\|8883\|8083\|8084\|18083'` 檢查端口 |
| Dashboard 無法訪問 | 容器尚未就緒 | 等待 30 秒後再試，用 `docker compose ps` 確認 healthy |
| MQTT 連接被拒 | 服務未啟動 | 確認容器運行中，檢查防火牆 |
| 密碼登入失敗 | 舊數據衝突 | `docker compose down -v && docker compose up -d` 清除重建 |

### 完全移除

```bash
# 停止並移除容器 + 網路
docker compose down

# 停止並移除容器 + 網路 + 所有數據
docker compose down -v
```

### 生產環境建議

1. **修改預設密碼**：`EMQX_DASHBOARD_PASSWORD` 設定強密碼
2. **關閉匿名連接**：`EMQX_ALLOW_ANONYMOUS=false`
3. **啟用 SSL/TLS**：配置 8883 端口的 TLS 證書
4. **防火牆規則**：僅開放必要端口給信任的 IP
5. **定期備份**：排程備份 emqx_data volume
6. **資源限制**：在 docker-compose.yml 中加入 `deploy.resources.limits`

---

## English

### Introduction

This project provides a complete one-click deployment solution for **EMQX MQTT Broker** using Docker Compose. EMQX is the world's leading open-source distributed MQTT message broker, designed for IoT, M2M, and mobile applications, supporting millions of concurrent connections.

This deployment has been verified on:
- **EMQX 6.0.0** (`emqx/emqx:latest`)
- **Podman 4.9.3** + podman-compose 1.0.6
- **Ubuntu Linux** (Docker also supported)

### Architecture

```
┌──────────────────────────────────────────────┐
│              EMQX Broker v6.0.0              │
│  ┌──────────────────────────────────────────┐│
│  │  MQTT TCP:       1883                    ││
│  │  MQTT SSL/TLS:   8883                    ││
│  │  WebSocket:      8083                    ││
│  │  WebSocket SSL:  8084                    ││
│  │  Dashboard UI:   18083                   ││
│  └──────────────────────────────────────────┘│
│  Volumes:                                    │
│  ├── emqx_data (config + runtime data)       │
│  └── emqx_log  (logs)                        │
└──────────────────────────────────────────────┘
               ▲
               │ MQTT Connections
     ┌─────────┴─────────┐
     │  IoT Devices /    │
     │  Sensors          │
     └───────────────────┘
```

### Requirements

| Item | Minimum |
|------|---------|
| Container Engine | Docker 20.10+ or Podman 4.0+ |
| Compose Tool | Docker Compose 2.0+ or podman-compose |
| Memory | 512MB+ (1GB+ recommended) |
| Ports | 1883, 8883, 8083, 8084, 18083 |

### Project File Structure

```
Woow_eqmx_docker_compose_all/
├── docker-compose.yml   # Main Docker Compose configuration
├── .env.example         # Environment variables template (copy to .env)
├── .gitignore           # Git ignore rules (excludes .env and sensitive files)
├── README.md            # This file - full bilingual deployment guide
└── DEPLOY_SKILL.md      # AI rapid deployment skill guide
```

### Quick Start

#### 1. Get the Project

```bash
git clone https://github.com/WOOWTECH/Woow_eqmx_docker_compose_all.git
cd Woow_eqmx_docker_compose_all
```

#### 2. Create Environment Config

```bash
cp .env.example .env
```

#### 3. Modify Settings (Recommended)

Edit `.env` file, at minimum change the Dashboard password:

```bash
nano .env
# or
vim .env
```

Key settings:

| Variable | Default | Description |
|----------|---------|-------------|
| `EMQX_VERSION` | latest | EMQX image version |
| `EMQX_DASHBOARD_USER` | admin | Dashboard login username |
| `EMQX_DASHBOARD_PASSWORD` | public | Dashboard login password (**change this**) |
| `EMQX_DASHBOARD_PORT` | 18083 | Dashboard web port |
| `MQTT_TCP_PORT` | 1883 | MQTT TCP port |
| `MQTT_SSL_PORT` | 8883 | MQTT SSL port |
| `MQTT_WS_PORT` | 8083 | MQTT WebSocket port |
| `MQTT_WSS_PORT` | 8084 | MQTT WebSocket SSL port |
| `EMQX_HOST` | 127.0.0.1 | Node host address |
| `EMQX_ALLOW_ANONYMOUS` | true | Allow anonymous connections |

#### 4. Start Services

**Using Docker Compose:**
```bash
docker compose up -d
```

**Using Podman:**
```bash
podman-compose up -d
```

#### 5. Verify Deployment

Wait ~30 seconds for EMQX to fully start:

```bash
# Check container status (STATUS should show healthy)
docker compose ps
# or
podman-compose ps

# Check EMQX node status
docker exec emqx emqx ctl status
# or
podman exec emqx emqx ctl status
# Expected: Node 'emqx@127.0.0.1' 6.0.0 is started

# Test Dashboard accessibility
curl -s -o /dev/null -w "%{http_code}" http://localhost:18083
# Expected: 200
```

#### 6. Access Dashboard

Open browser: **http://localhost:18083**

- Username: `admin`
- Password: `public` (or the password set in `.env`)

### Port Reference

| Port | Protocol | Purpose | Default |
|------|----------|---------|---------|
| 1883 | MQTT TCP | Standard MQTT connection | Enabled |
| 8883 | MQTT SSL | TLS encrypted MQTT | Enabled |
| 8083 | WebSocket | MQTT over WebSocket | Enabled |
| 8084 | WebSocket SSL | MQTT over WSS | Enabled |
| 18083 | HTTP | Dashboard management UI | Enabled |

### Common Commands

```bash
# ===== Service Management =====
# Start
docker compose up -d

# Stop
docker compose down

# Restart
docker compose restart

# View live logs
docker compose logs -f emqx

# ===== EMQX Management =====
# Enter EMQX container
docker compose exec emqx sh

# Check node status
docker compose exec emqx emqx ctl status

# List connected clients
docker compose exec emqx emqx ctl clients list

# List all topics
docker compose exec emqx emqx ctl topics list

# View cluster info
docker compose exec emqx emqx ctl cluster status
```

> **Podman users**: Replace `docker compose` with `podman-compose` and `docker exec` with `podman exec`.

### Testing MQTT Connection

Using mosquitto client tools:

```bash
# Install mosquitto client
# Ubuntu/Debian
sudo apt install mosquitto-clients
# macOS
brew install mosquitto
# Alpine
apk add mosquitto-clients

# Subscribe in Terminal 1
mosquitto_sub -h localhost -p 1883 -t "test/topic" -v

# Publish in Terminal 2
mosquitto_pub -h localhost -p 1883 -t "test/topic" -m "Hello EMQX!"

# Authenticated connection (when anonymous disabled)
mosquitto_sub -h localhost -p 1883 -t "test/#" -u "username" -P "password" -v
mosquitto_pub -h localhost -p 1883 -t "test/topic" -u "username" -P "password" -m "Hello!"
```

### Data Persistence

EMQX data is stored in Docker/Podman volumes, preserved across container removal:

- **emqx_data** - Configuration, rule engine, authentication data
- **emqx_log** - Runtime logs

#### Backup

```bash
# Backup data
docker run --rm -v emqx_data:/data -v $(pwd):/backup alpine tar czf /backup/emqx_data_backup.tar.gz /data

# Backup logs
docker run --rm -v emqx_log:/data -v $(pwd):/backup alpine tar czf /backup/emqx_log_backup.tar.gz /data
```

#### Restore

```bash
# Restore data
docker run --rm -v emqx_data:/data -v $(pwd):/backup alpine tar xzf /backup/emqx_data_backup.tar.gz -C /

# Restore logs
docker run --rm -v emqx_log:/data -v $(pwd):/backup alpine tar xzf /backup/emqx_log_backup.tar.gz -C /
```

### Troubleshooting

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| Container keeps restarting | Port conflict | `ss -tlnp \| grep -E '1883\|8883\|8083\|8084\|18083'` |
| Dashboard unreachable | Container not ready | Wait 30s, check `docker compose ps` for healthy |
| MQTT connection refused | Service not started | Confirm container running, check firewall |
| Login fails | Stale data | `docker compose down -v && docker compose up -d` |

### Complete Removal

```bash
# Stop and remove container + network
docker compose down

# Stop and remove container + network + all data
docker compose down -v
```

### Production Recommendations

1. **Change default password**: Set a strong `EMQX_DASHBOARD_PASSWORD`
2. **Disable anonymous access**: `EMQX_ALLOW_ANONYMOUS=false`
3. **Enable SSL/TLS**: Configure TLS certificates for port 8883
4. **Firewall rules**: Only open necessary ports to trusted IPs
5. **Regular backups**: Schedule emqx_data volume backups
6. **Resource limits**: Add `deploy.resources.limits` in docker-compose.yml
