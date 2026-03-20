# EMQX K3s/Kubernetes 部署指南

[English](#english) | [中文](#中文)

---

## English

### Overview

Enterprise-grade MQTT message broker for IoT (Internet of Things). EMQX supports MQTT 5.0, MQTT-SN, CoAP, and WebSocket protocols, capable of handling millions of concurrent connections with low latency. It provides a built-in web dashboard for monitoring, rule engine for data routing, and authentication/ACL for device access control.

> **GitHub Repo (Podman/Docker):** [Woow_eqmx_docker_compose_all](https://github.com/WOOWTECH/Woow_eqmx_docker_compose_all)

### Architecture

```
                           EMQX K3s Architecture
 ============================================================================

   External Access                 K3s Cluster (namespace: emqx)
  +----------------+          +--------------------------------------------+
  |                |          |                                            |
  |  MQTT Client   |          |   +------------------------------------+   |
  |  :31883  ------+---NodePort-->|  Service: emqx (MQTT)              |   |
  |                |          |   |  ClusterIP :1883                   |   |
  +----------------+          |   +----------------+-------------------+   |
                              |                    |                       |
  +----------------+          |                    v                       |
  |                |          |   +------------------------------------+   |
  |  Browser       |          |   |  Pod: emqx (StatefulSet)           |   |
  |  :31808  ------+---NodePort-->|  Image: emqx/emqx                  |   |
  |  (Dashboard)   |          |   |                                    |   |
  +----------------+          |   |  Ports:                            |   |
                              |   |    1883  - MQTT TCP                |   |
                              |   |    8883  - MQTT TLS                |   |
                              |   |    8083  - MQTT WebSocket          |   |
                              |   |    8084  - MQTT WebSocket TLS      |   |
                              |   |    18083 - Dashboard Web UI        |   |
                              |   |                                    |   |
                              |   |  Volumes:                          |   |
                              |   |    /opt/emqx/data (5Gi)            |   |
                              |   |    /opt/emqx/log  (2Gi)            |   |
                              |   +------------------------------------+   |
                              |                                            |
                              |   Headless Service:                        |
                              |   emqx-0.emqx-headless.emqx.svc           |
                              |   (Erlang node clustering)                 |
                              +--------------------------------------------+
```

### Features

- MQTT 5.0, MQTT-SN, CoAP, and WebSocket protocol support
- Millions of concurrent connections with low latency
- Built-in web dashboard for monitoring and management
- Rule engine for data routing and transformation
- Authentication and ACL for device access control
- Erlang-based clustering with headless service DNS

### Quick Start

```bash
# 1. Update secrets before deploying
nano k8s-manifests/emqx/secret.yaml

# 2. Deploy EMQX
kubectl apply -k k8s-manifests/emqx/

# 3. Verify pods are running
kubectl -n emqx get pods

# 4. Watch EMQX startup logs
kubectl -n emqx logs sts/emqx -f
```

### Configuration

#### Environment Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `EMQX_NODE_NAME` | Erlang node name (FQDN for clustering) | `emqx@emqx-0.emqx-headless.emqx.svc.cluster.local` | Yes |
| `EMQX_ALLOW_ANONYMOUS` | Allow connections without authentication | `true` | No |

#### Secrets

Edit `secret.yaml` before deploying:

| Secret Key | Description | Default (change me!) |
|------------|-------------|----------------------|
| `EMQX_DASHBOARD__DEFAULT_USERNAME` | Dashboard admin username | `admin` |
| `EMQX_DASHBOARD__DEFAULT_PASSWORD` | Dashboard admin password | `changeme_emqx_2024` |

```bash
nano k8s-manifests/emqx/secret.yaml
```

### Accessing the Service

| Endpoint | URL / Port | Protocol |
|----------|-----------|----------|
| MQTT TCP | `<node-ip>:31883` | MQTT (NodePort) |
| MQTT SSL/TLS | `emqx.emqx.svc.cluster.local:8883` | MQTTS (ClusterIP) |
| MQTT WebSocket | `emqx.emqx.svc.cluster.local:8083` | WS (ClusterIP) |
| MQTT WebSocket SSL | `emqx.emqx.svc.cluster.local:8084` | WSS (ClusterIP) |
| Dashboard Web UI | `http://<node-ip>:31808` | HTTP (NodePort) |

#### Port Summary

| Port | Protocol | NodePort | Description |
|------|----------|----------|-------------|
| 1883 | TCP | 31883 | MQTT standard |
| 8883 | TCP | - | MQTT over TLS |
| 8083 | TCP | - | MQTT over WebSocket |
| 8084 | TCP | - | MQTT over WebSocket + TLS |
| 18083 | TCP | 31808 | Dashboard Web UI |

Default dashboard credentials: `admin` / `changeme_emqx_2024`

### Data Persistence

| PVC Name | Mount Path | Size | Purpose |
|----------|------------|------|---------|
| `emqx-data` | `/opt/emqx/data` | 5Gi | MQTT retained messages, sessions, configuration |
| `emqx-log` | `/opt/emqx/log` | 2Gi | EMQX log files |

All PVCs use the `local-path` storage class (k3s default).

### Backup & Restore

#### Backup

```bash
# 1. Backup EMQX data (retained messages, authentication data, ACLs)
kubectl -n emqx exec sts/emqx -- tar czf /tmp/emqx-data-backup.tar.gz /opt/emqx/data
kubectl -n emqx cp emqx/emqx-0:/tmp/emqx-data-backup.tar.gz ./emqx-data-backup.tar.gz

# 2. Export EMQX configuration
kubectl -n emqx exec sts/emqx -- cat /opt/emqx/data/configs/cluster.hocon > emqx-cluster-config.hocon
```

#### Restore

```bash
# 1. Restore EMQX data
kubectl -n emqx cp ./emqx-data-backup.tar.gz emqx/emqx-0:/tmp/emqx-data-backup.tar.gz
kubectl -n emqx exec sts/emqx -- tar xzf /tmp/emqx-data-backup.tar.gz -C /

# 2. Restart EMQX
kubectl -n emqx rollout restart sts/emqx
```

### Useful Commands

```bash
# Check pod status
kubectl -n emqx get pods

# View real-time logs
kubectl -n emqx logs sts/emqx -f

# View all services and ports
kubectl -n emqx get svc

# Restart EMQX
kubectl -n emqx rollout restart sts/emqx

# Test MQTT connectivity
telnet <node-ip> 31883

# Export cluster configuration
kubectl -n emqx exec sts/emqx -- cat /opt/emqx/data/configs/cluster.hocon
```

### Troubleshooting

#### EMQX dashboard not accessible

```bash
# Check pod status
kubectl -n emqx get pods
kubectl -n emqx logs sts/emqx

# Verify services
kubectl -n emqx get svc
```

#### MQTT clients cannot connect

1. Verify the NodePort is accessible: `telnet <node-ip> 31883`
2. Check if anonymous access is enabled (default: `true`)
3. For authenticated access, configure users in the EMQX dashboard under Authentication

#### Disable anonymous access for production

Set `EMQX_ALLOW_ANONYMOUS` to `false` in `configmap.yaml` and configure authentication in the dashboard:

```bash
kubectl apply -k k8s-manifests/emqx/
kubectl -n emqx rollout restart sts/emqx
```

#### High memory usage

EMQX stores sessions in memory. If memory usage is high, check the number of connected clients in the dashboard and adjust session expiry settings.

#### Cluster node name issues

The node name must match the pod's DNS name. The default uses the headless service FQDN. If renaming, update `EMQX_NODE_NAME` in `configmap.yaml`.

### File Structure

```
emqx/
├── configmap.yaml            # Environment variables (node name, anonymous access)
├── emqx-service.yaml         # NodePort services (31883 -> 1883, 31808 -> 18083)
├── emqx-statefulset.yaml     # StatefulSet for EMQX broker pod
├── kustomization.yaml        # Kustomize resource list
├── namespace.yaml            # Namespace: emqx
├── pvc.yaml                  # PersistentVolumeClaims for data and logs
├── README.md                 # This file
└── secret.yaml               # Dashboard credentials (change before deploy!)
```

---

## 中文

### 概述

EMQX 是企業級 MQTT 訊息代理伺服器，專為物聯網（IoT）設計。支援 MQTT 5.0、MQTT-SN、CoAP 及 WebSocket 協定，能以低延遲處理數百萬並發連線。內建 Web 儀表板用於監控、規則引擎用於資料路由，以及身份驗證/ACL 用於裝置存取控制。

> **GitHub Repo (Podman/Docker):** [Woow_eqmx_docker_compose_all](https://github.com/WOOWTECH/Woow_eqmx_docker_compose_all)

### 架構圖

```
                           EMQX K3s 架構
 ============================================================================

   外部存取                      K3s 叢集 (namespace: emqx)
  +----------------+          +--------------------------------------------+
  |                |          |                                            |
  |  MQTT 用戶端   |          |   +------------------------------------+   |
  |  :31883  ------+---NodePort-->|  Service: emqx (MQTT)              |   |
  |                |          |   |  ClusterIP :1883                   |   |
  +----------------+          |   +----------------+-------------------+   |
                              |                    |                       |
  +----------------+          |                    v                       |
  |                |          |   +------------------------------------+   |
  |  瀏覽器        |          |   |  Pod: emqx (StatefulSet)           |   |
  |  :31808  ------+---NodePort-->|  映像: emqx/emqx                   |   |
  |  (儀表板)      |          |   |                                    |   |
  +----------------+          |   |  埠號:                             |   |
                              |   |    1883  - MQTT TCP                |   |
                              |   |    8883  - MQTT TLS                |   |
                              |   |    8083  - MQTT WebSocket          |   |
                              |   |    8084  - MQTT WebSocket TLS      |   |
                              |   |    18083 - 儀表板 Web UI            |   |
                              |   |                                    |   |
                              |   |  磁碟區:                           |   |
                              |   |    /opt/emqx/data (5Gi)            |   |
                              |   |    /opt/emqx/log  (2Gi)            |   |
                              |   +------------------------------------+   |
                              |                                            |
                              |   Headless Service:                        |
                              |   emqx-0.emqx-headless.emqx.svc           |
                              |   (Erlang 節點叢集)                        |
                              +--------------------------------------------+
```

### 功能特色

- 支援 MQTT 5.0、MQTT-SN、CoAP 及 WebSocket 協定
- 數百萬並發連線，低延遲處理
- 內建 Web 儀表板用於監控與管理
- 規則引擎用於資料路由與轉換
- 身份驗證與 ACL 裝置存取控制
- 基於 Erlang 的叢集，使用 Headless Service DNS

### 快速開始

```bash
# 1. 部署前先更新密鑰
nano k8s-manifests/emqx/secret.yaml

# 2. 部署 EMQX
kubectl apply -k k8s-manifests/emqx/

# 3. 確認 Pod 運作中
kubectl -n emqx get pods

# 4. 查看 EMQX 啟動日誌
kubectl -n emqx logs sts/emqx -f
```

### 設定

#### 環境變數

| 變數 | 說明 | 預設值 | 必填 |
|------|------|--------|------|
| `EMQX_NODE_NAME` | Erlang 節點名稱（叢集用 FQDN） | `emqx@emqx-0.emqx-headless.emqx.svc.cluster.local` | 是 |
| `EMQX_ALLOW_ANONYMOUS` | 允許無身份驗證連線 | `true` | 否 |

#### Secrets（密鑰）

部署前請編輯 `secret.yaml`：

| Secret Key | 說明 | 預設值（請變更！） |
|------------|------|-------------------|
| `EMQX_DASHBOARD__DEFAULT_USERNAME` | 儀表板管理員帳號 | `admin` |
| `EMQX_DASHBOARD__DEFAULT_PASSWORD` | 儀表板管理員密碼 | `changeme_emqx_2024` |

```bash
nano k8s-manifests/emqx/secret.yaml
```

### 存取服務

| 端點 | URL / 埠號 | 協定 |
|------|-----------|------|
| MQTT TCP | `<node-ip>:31883` | MQTT (NodePort) |
| MQTT SSL/TLS | `emqx.emqx.svc.cluster.local:8883` | MQTTS (ClusterIP) |
| MQTT WebSocket | `emqx.emqx.svc.cluster.local:8083` | WS (ClusterIP) |
| MQTT WebSocket SSL | `emqx.emqx.svc.cluster.local:8084` | WSS (ClusterIP) |
| 儀表板 Web UI | `http://<node-ip>:31808` | HTTP (NodePort) |

#### 埠號總覽

| 埠號 | 協定 | NodePort | 說明 |
|------|------|----------|------|
| 1883 | TCP | 31883 | MQTT 標準 |
| 8883 | TCP | - | MQTT over TLS |
| 8083 | TCP | - | MQTT over WebSocket |
| 8084 | TCP | - | MQTT over WebSocket + TLS |
| 18083 | TCP | 31808 | 儀表板 Web UI |

預設儀表板帳號密碼：`admin` / `changeme_emqx_2024`

### 資料持久化

| PVC 名稱 | 掛載路徑 | 大小 | 用途 |
|----------|----------|------|------|
| `emqx-data` | `/opt/emqx/data` | 5Gi | MQTT 保留訊息、工作階段、設定 |
| `emqx-log` | `/opt/emqx/log` | 2Gi | EMQX 日誌檔案 |

所有 PVC 使用 `local-path` 儲存類別（k3s 預設）。

### 備份與還原

#### 備份

```bash
# 1. 備份 EMQX 資料（保留訊息、認證資料、ACL）
kubectl -n emqx exec sts/emqx -- tar czf /tmp/emqx-data-backup.tar.gz /opt/emqx/data
kubectl -n emqx cp emqx/emqx-0:/tmp/emqx-data-backup.tar.gz ./emqx-data-backup.tar.gz

# 2. 匯出 EMQX 設定
kubectl -n emqx exec sts/emqx -- cat /opt/emqx/data/configs/cluster.hocon > emqx-cluster-config.hocon
```

#### 還原

```bash
# 1. 還原 EMQX 資料
kubectl -n emqx cp ./emqx-data-backup.tar.gz emqx/emqx-0:/tmp/emqx-data-backup.tar.gz
kubectl -n emqx exec sts/emqx -- tar xzf /tmp/emqx-data-backup.tar.gz -C /

# 2. 重啟 EMQX
kubectl -n emqx rollout restart sts/emqx
```

### 實用指令

```bash
# 查看 Pod 狀態
kubectl -n emqx get pods

# 查看即時日誌
kubectl -n emqx logs sts/emqx -f

# 查看所有服務與埠號
kubectl -n emqx get svc

# 重啟 EMQX
kubectl -n emqx rollout restart sts/emqx

# 測試 MQTT 連線
telnet <node-ip> 31883

# 匯出叢集設定
kubectl -n emqx exec sts/emqx -- cat /opt/emqx/data/configs/cluster.hocon
```

### 疑難排解

#### EMQX 儀表板無法存取

```bash
# 檢查 Pod 狀態
kubectl -n emqx get pods
kubectl -n emqx logs sts/emqx

# 確認服務
kubectl -n emqx get svc
```

#### MQTT 用戶端無法連線

1. 確認 NodePort 可存取：`telnet <node-ip> 31883`
2. 檢查是否啟用匿名存取（預設：`true`）
3. 若需認證存取，在 EMQX 儀表板的 Authentication 中設定使用者

#### 正式環境停用匿名存取

在 `configmap.yaml` 中將 `EMQX_ALLOW_ANONYMOUS` 設為 `false`，並在儀表板中設定認證：

```bash
kubectl apply -k k8s-manifests/emqx/
kubectl -n emqx rollout restart sts/emqx
```

#### 高記憶體使用

EMQX 將工作階段儲存在記憶體中。若記憶體使用偏高，請在儀表板中檢查已連線用戶端數量，並調整工作階段過期設定。

#### 叢集節點名稱問題

節點名稱必須與 Pod 的 DNS 名稱一致。預設使用 Headless Service FQDN。若需重新命名，請更新 `configmap.yaml` 中的 `EMQX_NODE_NAME`。

### 檔案結構

```
emqx/
├── configmap.yaml            # 環境變數（節點名稱、匿名存取）
├── emqx-service.yaml         # NodePort 服務 (31883 -> 1883, 31808 -> 18083)
├── emqx-statefulset.yaml     # EMQX 代理 Pod 的 StatefulSet
├── kustomization.yaml        # Kustomize 資源列表
├── namespace.yaml            # 命名空間: emqx
├── pvc.yaml                  # 持久卷宣告（資料與日誌）
├── README.md                 # 本文件
└── secret.yaml               # 儀表板帳號密碼（部署前請變更！）
```
