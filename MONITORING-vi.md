# PostgreSQL HA Cluster - Hệ thống Giám sát

🇬🇧 [English Version](MONITORING.md)

## Tổng quan

Hệ thống giám sát đầy đủ với **Prometheus + Grafana** để theo dõi PostgreSQL HA cluster.

## Kiến trúc

```
┌─────────────────────────────────────────────────────────────┐
│                    Monitoring Stack                          │
│                                                              │
│  ┌──────────────┐         ┌──────────────┐                 │
│  │  Prometheus  │────────▶│   Grafana    │                 │
│  │   :9090      │         │    :3000     │                 │
│  └──────┬───────┘         └──────────────┘                 │
│         │                                                    │
│         │ scrape metrics                                     │
│         │                                                    │
└─────────┼────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────┐
│              PostgreSQL HA Cluster (3 nodes)                 │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Node 1 (pg-node1)                                    │   │
│  │  • node_exporter:9100      (system metrics)         │   │
│  │  • postgres_exporter:9187  (database metrics)       │   │
│  │  • pgbouncer_exporter:9127 (pool metrics)           │   │
│  │  • patroni:8008/metrics    (HA metrics)             │   │
│  │  • etcd:2379/metrics       (DCS metrics)            │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Node 2 (pg-node2) - Các exporter tương tự          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ Node 3 (pg-node3) - Các exporter tương tự          │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## Thành phần

### 1. **Prometheus** (Port 9090)

- Cơ sở dữ liệu time-series
- Thu thập metrics mỗi 15 giây
- Lưu trữ 30 ngày (có thể cấu hình)
- Quy tắc cảnh báo cho sự kiện nghiêm trọng

### 2. **Grafana** (Port 3000)

- Dashboard trực quan hóa
- Dashboard cấu hình sẵn:
  - Tổng quan PostgreSQL
  - Patroni HA Cluster
  - Metrics Hệ thống (Node Exporter)
  - etcd Cluster
- Thông báo cảnh báo (tùy chọn)

### 3. **Exporters** (Trên mỗi PostgreSQL node)

- **node_exporter** (9100): CPU, RAM, Disk, Network
- **postgres_exporter** (9187): Kết nối, truy vấn, replication lag
- **pgbouncer_exporter** (9127): Thống kê connection pool
- **patroni metrics** (8008): Trạng thái leader/replica, sự kiện failover
- **etcd metrics** (2379): Sức khỏe cluster, thay đổi leader

## Cài đặt

### Bước 1: Cấu hình Môi trường

```bash
cd /path/to/postgres-patroni-etcd-install

# Sao chép và chỉnh sửa .env
cp .env.example .env
nano .env
```

**Biến monitoring chính trong `.env`:**

```bash
# Bật/Tắt Monitoring
MONITORING_ENABLED=true

# Monitoring Server (có thể là node riêng hoặc dùng 1 trong 3 postgres nodes)
MONITORING_SERVER_IP=10.0.0.22
MONITORING_SERVER_NAME=pg-node1

# Prometheus
PROMETHEUS_VERSION=3.0.1
PROMETHEUS_PORT=9090
PROMETHEUS_RETENTION_TIME=30d
PROMETHEUS_RETENTION_SIZE=50GB

# Grafana
GRAFANA_PORT=3000
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=ChangeMe@AdminPass#2024

# Exporters
NODE_EXPORTER_PORT=9100
POSTGRES_EXPORTER_PORT=9187
PGBOUNCER_EXPORTER_PORT=9127
```

**Lưu ý:**

- Set `MONITORING_ENABLED=true` để bật monitoring
- `MONITORING_SERVER_IP` và `MONITORING_SERVER_NAME` để chỉ định máy chủ monitoring riêng
- Nếu không có server riêng, có thể dùng `pg-node1` (hoặc node nào có tài nguyên dư)

### Bước 2: Triển khai Monitoring Stack

```bash
# Nạp biến môi trường
set -a && source .env && set +a

# Triển khai toàn bộ (khuyến nghị)
./scripts/deploy_monitoring.sh --all

# Hoặc triển khai từng thành phần
./scripts/deploy_monitoring.sh --exporters    # Chỉ exporters
./scripts/deploy_monitoring.sh --prometheus   # Chỉ Prometheus
./scripts/deploy_monitoring.sh --grafana      # Chỉ Grafana
```

### Bước 3: Xác minh Triển khai

```bash
# Kiểm tra sức khỏe tất cả services
./scripts/deploy_monitoring.sh --check
```

Kết quả mong đợi:

```
✓ Prometheus is healthy
✓ Grafana is healthy
✓ Node Exporter is running (all nodes)
✓ PostgreSQL Exporter is running (all nodes)
✓ PgBouncer Exporter is running (all nodes)
```

## URL Truy cập

### Prometheus

```
URL: http://<node-ip>:9090
Targets: http://<node-ip>:9090/targets
Alerts: http://<node-ip>:9090/alerts
```

### Grafana

```
URL: http://<node-ip>:3000
Username: admin (mặc định)
Password: Xem GRAFANA_ADMIN_PASSWORD trong .env
```

### Exporters (mỗi node)

```
Node Exporter: http://<node-ip>:9100/metrics
PostgreSQL Exporter: http://<node-ip>:9187/metrics
PgBouncer Exporter: http://<node-ip>:9127/metrics
Patroni Metrics: http://<node-ip>:8008/metrics
etcd Metrics: http://<node-ip>:2379/metrics
```

## Grafana Dashboards

Sau khi đăng nhập Grafana, có 4 dashboard cấu hình sẵn:

### 1. **Tổng quan PostgreSQL**

- Trạng thái database (hoạt động/ngừng)
- Kết nối đang hoạt động theo database
- Replication lag
- Tỷ lệ transaction (commits/rollbacks)
- Tỷ lệ cache hit
- Số lượng dead tuples

### 2. **Patroni HA Cluster**

- Node leader hiện tại
- Trạng thái thành viên cluster
- Thay đổi timeline (sự kiện failover)
- Kết nối DCS (etcd)

### 3. **Node Exporter - Metrics Hệ thống**

- Sử dụng CPU theo core
- Sử dụng bộ nhớ (tổng/khả dụng)
- Sử dụng disk theo mount point
- Lưu lượng mạng (RX/TX)
- Disk I/O

### 4. **etcd Cluster**

- Trạng thái leader
- Tỷ lệ thay đổi leader
- Lưu lượng RPC
- Thời gian đồng bộ disk

## Quy tắc Cảnh báo

Prometheus có sẵn quy tắc cảnh báo cho:

### Cảnh báo Nghiêm trọng (Critical)

- **PostgreSQLDown**: Database instance ngừng > 1 phút
- **PatroniNoLeader**: Cluster không có leader > 1 phút
- **EtcdNoLeader**: etcd không có leader > 1 phút
- **NodeDown**: Server không phản hồi > 2 phút
- **PgBouncerDown**: Connection pooler ngừng > 2 phút

### Cảnh báo (Warning)

- **PostgreSQLReplicationLag**: Lag > 60 giây
- **PostgreSQLTooManyConnections**: > 80% max connections
- **HighCPUUsage**: CPU > 80% trong 5 phút
- **HighMemoryUsage**: RAM > 85% trong 5 phút
- **LowDiskSpace**: Disk < 15% dung lượng trống

## Bảo trì

### Cập nhật Exporters

```bash
# Cập nhật phiên bản trong .env
nano .env
# Thay đổi NODE_EXPORTER_VERSION, POSTGRES_EXPORTER_VERSION, v.v.

# Triển khai lại
set -a && source .env && set +a
./scripts/deploy_monitoring.sh --exporters
```

### Khởi động lại Services

```bash
# Prometheus
ssh root@<node-ip> "systemctl restart prometheus"

# Grafana
ssh root@<node-ip> "systemctl restart grafana-server"

# Exporters (mỗi node)
ssh root@<node-ip> "systemctl restart node_exporter"
ssh root@<node-ip> "systemctl restart postgres_exporter"
ssh root@<node-ip> "systemctl restart pgbouncer_exporter"
```

### Kiểm tra Logs

```bash
# Prometheus
ssh root@<node-ip> "journalctl -u prometheus -f"

# Grafana
ssh root@<node-ip> "journalctl -u grafana-server -f"

# Exporters
ssh root@<node-ip> "journalctl -u node_exporter -f"
ssh root@<node-ip> "journalctl -u postgres_exporter -f"
```

## Xử lý Sự cố

### Prometheus không thu thập được metrics

```bash
# Kiểm tra firewall
ufw status

# Mở port nếu cần
ufw allow 9100/tcp  # node_exporter
ufw allow 9187/tcp  # postgres_exporter
ufw allow 9127/tcp  # pgbouncer_exporter
```

### PostgreSQL Exporter không kết nối được

```bash
# Kiểm tra DSN trong /etc/default/postgres_exporter
cat /etc/default/postgres_exporter

# Test kết nối thủ công
psql "postgresql://admin:<password>@localhost:5432/postgres"

# Khởi động lại exporter
systemctl restart postgres_exporter
```

### Grafana không hiện dashboards

```bash
# Kiểm tra thư mục provisioning
ls -la /var/lib/grafana/dashboards/

# Kiểm tra Grafana logs
journalctl -u grafana-server -n 100
```

## Tác động Hiệu năng

Hệ thống monitoring có tác động tối thiểu:

| Thành phần | CPU | RAM | Disk I/O |
|------------|-----|-----|----------|
| Prometheus | <2% | ~1GB | Thấp |
| Grafana | <1% | ~200MB | Rất thấp |
| node_exporter | <0.5% | ~20MB | Rất thấp |
| postgres_exporter | <1% | ~50MB | Thấp |
| pgbouncer_exporter | <0.5% | ~20MB | Rất thấp |

**Tổng overhead mỗi node:** ~2-3% CPU, ~100MB RAM

## Khuyến nghị Bảo mật

1. **Đổi mật khẩu Grafana mặc định** trong `.env`
2. **Bật xác thực** cho Prometheus nếu expose ra internet
3. **Dùng firewall** để giới hạn truy cập đến monitoring ports
4. **Bật SSL/TLS** cho Grafana trong production
5. **Luân chuyển secret** trong `GRAFANA_SECRET_KEY`

## Tích hợp Alertmanager (Tùy chọn)

Nếu muốn gửi cảnh báo qua Slack/Email:

```bash
# Cài đặt Alertmanager
# Set PROMETHEUS_ALERTMANAGER_TARGETS trong .env
PROMETHEUS_ALERTMANAGER_TARGETS=localhost:9093

# Triển khai lại Prometheus
./scripts/deploy_monitoring.sh --prometheus
```

## Cấu trúc File

```
roles/
├── prometheus/
│   ├── tasks/main.yml
│   ├── templates/
│   │   ├── prometheus.yml.j2
│   │   ├── prometheus.service.j2
│   │   └── alert_rules.yml.j2
│   ├── handlers/main.yml
│   └── defaults/main.yml
│
├── grafana/
│   ├── tasks/main.yml
│   ├── templates/
│   │   ├── grafana.ini.j2
│   │   ├── datasource.yml.j2
│   │   ├── dashboard_provisioning.yml.j2
│   │   └── dashboards/
│   │       ├── postgresql_dashboard.json.j2
│   │       ├── patroni_dashboard.json.j2
│   │       ├── node_exporter_dashboard.json.j2
│   │       └── etcd_dashboard.json.j2
│   ├── handlers/main.yml
│   └── defaults/main.yml
│
└── exporters/
    ├── tasks/main.yml
    ├── templates/
    │   ├── node_exporter.service.j2
    │   ├── postgres_exporter.service.j2
    │   ├── postgres_exporter.env.j2
    │   ├── pgbouncer_exporter.service.j2
    │   └── pgbouncer_exporter.env.j2
    ├── handlers/main.yml
    └── defaults/main.yml
```

## Tài liệu Tham khảo

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [PostgreSQL Exporter](https://github.com/prometheus-community/postgres_exporter)
- [Node Exporter](https://github.com/prometheus/node_exporter)
- [PgBouncer Exporter](https://github.com/prometheus-community/pgbouncer_exporter)
