# xScan Monitoring Stack - วิธีใช้งาน

ระบบ Monitoring ประกอบด้วย **Prometheus**, **Grafana**, และ **Blackbox Exporter** สำหรับตรวจสอบสถานะเว็บไซต์และเครือข่าย

---

## 📁 โครงสร้างโปรเจกต์

```
xScan/
├── docker-compose.yml                          # กำหนด services ทั้งหมด
├── prometheus/
│   └── prometheus.yml                          # Prometheus config & scrape targets
├── blackbox/
│   └── config.yml                              # Blackbox Exporter probe modules
├── grafana/
│   └── provisioning/
│       └── datasources/
│           └── datasource.yml                  # Auto-provision Prometheus datasource
└── doc/
    └── use-read.md                             # เอกสารนี้
```

---

## 🚀 เริ่มต้นใช้งาน

### ข้อกำหนดเบื้องต้น

- [Docker](https://docs.docker.com/get-docker/) ติดตั้งแล้ว
- [Docker Compose](https://docs.docker.com/compose/install/) v2 ขึ้นไป

### สั่งรันทุก Services

```bash
docker compose up -d
```

### ตรวจสอบสถานะ Services

```bash
docker compose ps
```

### ดู Logs

```bash
# ดู logs ทุก services
docker compose logs -f

# ดู logs เฉพาะ service
docker compose logs -f prometheus
docker compose logs -f grafana
docker compose logs -f blackbox_exporter
```

### หยุด Services

```bash
docker compose down
```

### หยุดและลบ Volume data ทั้งหมด

```bash
docker compose down -v
```

---

## 🌐 เข้าถึง Services

| Service            | URL                      | รายละเอียด                          |
| ------------------ | ------------------------ | ----------------------------------- |
| **Prometheus**     | http://localhost:9090     | Query metrics, ดู targets & alerts  |
| **Grafana**        | http://localhost:3000     | Dashboard สำหรับ visualization      |
| **Blackbox Exporter** | http://localhost:9115  | ดู probe results & metrics          |

---

## 🔑 Grafana Login

| ค่า       | ค่าเริ่มต้น |
| --------- | ----------- |
| Username  | `admin`     |
| Password  | `admin`     |

> ⚠️ **แนะนำ**: เปลี่ยนรหัสผ่านหลังจาก login ครั้งแรก หรือแก้ไขในไฟล์ `docker-compose.yml` ก่อน deploy

---

## ⚙️ การตั้งค่า

### เพิ่ม HTTP Probe Targets

แก้ไขไฟล์ `prometheus/prometheus.yml` ในส่วน `blackbox_http`:

```yaml
- job_name: "blackbox_http"
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets:
        - https://www.google.com
        - https://www.github.com
        - https://your-website.com       # <-- เพิ่ม URL ที่ต้องการตรวจสอบ
```

หลังแก้ไขแล้ว reload Prometheus:

```bash
docker compose restart prometheus
```

### เพิ่ม ICMP Probe Targets

แก้ไขไฟล์ `prometheus/prometheus.yml` ในส่วน `blackbox_icmp`:

```yaml
- job_name: "blackbox_icmp"
  metrics_path: /probe
  params:
    module: [icmp]
  static_configs:
    - targets:
        - 8.8.8.8
        - 1.1.1.1
        - 192.168.1.1                   # <-- เพิ่ม IP ที่ต้องการ ping
```

### Blackbox Exporter Modules ที่พร้อมใช้งาน

| Module          | ประเภท | คำอธิบาย                              |
| --------------- | ------ | ------------------------------------- |
| `http_2xx`      | HTTP   | ตรวจสอบ HTTP GET ว่าได้ status 200/301/302 |
| `http_post_2xx` | HTTP   | ตรวจสอบ HTTP POST                     |
| `tcp_connect`   | TCP    | ตรวจสอบว่า port เปิดอยู่               |
| `icmp`          | ICMP   | Ping เช็คว่า host ตอบสนองหรือไม่       |
| `dns_lookup`    | DNS    | ตรวจสอบ DNS resolution                |

---

## 📊 Grafana Dashboard แนะนำ

หลังจาก login เข้า Grafana แล้ว สามารถ import dashboard สำเร็จรูปได้:

1. ไปที่ **Dashboards** → **Import**
2. กรอก Dashboard ID แล้วกด **Load**

| Dashboard ID | ชื่อ                              | คำอธิบาย                          |
| ------------ | --------------------------------- | --------------------------------- |
| `7587`       | Prometheus Blackbox Exporter      | แสดงผล HTTP/ICMP probe results    |
| `1860`       | Node Exporter Full                | สำหรับ server monitoring (ต้องเพิ่ม node exporter) |
| `3662`       | Prometheus 2.0 Overview           | ภาพรวมของ Prometheus              |

---

## 🔍 Prometheus Query ที่มีประโยชน์

ใช้ใน Prometheus UI (http://localhost:9090) หรือ Grafana Explore:

```promql
# ตรวจสอบว่า target ขึ้นหรือไม่ (1 = up, 0 = down)
probe_success

# เวลาที่ใช้ในการ probe (วินาที)
probe_duration_seconds

# HTTP status code ที่ได้รับ
probe_http_status_code

# ตรวจสอบ SSL certificate หมดอายุ (วินาที)
probe_ssl_earliest_cert_expiry - time()

# ดู uptime ทุก targets
up
```

---

## 🛠️ การแก้ไขปัญหา

### Prometheus ขึ้น Target เป็น DOWN

```bash
# ตรวจสอบ config ถูกต้อง
docker compose exec prometheus promtool check config /etc/prometheus/prometheus.yml

# ดู logs
docker compose logs prometheus
```

### Blackbox Exporter ไม่ทำงาน

```bash
# ทดสอบ probe ด้วยมือ
curl "http://localhost:9115/probe?target=https://www.google.com&module=http_2xx"

# ดู logs
docker compose logs blackbox_exporter
```

### Grafana ไม่เห็น Datasource

```bash
# ตรวจสอบว่า provisioning mount ถูกต้อง
docker compose exec grafana ls -la /etc/grafana/provisioning/datasources/

# Restart grafana
docker compose restart grafana
```

---

## 📝 หมายเหตุ

- Prometheus เก็บข้อมูลไว้ **30 วัน** (ปรับได้ใน `docker-compose.yml` ที่ `--storage.tsdb.retention.time`)
- ข้อมูลถูกเก็บใน Docker volumes: `prometheus_data` และ `grafana_data`
- ทุก services ตั้ง `restart: always` จะเริ่มต้นอัตโนมัติเมื่อ Docker restart
- ทุก services อยู่ใน network `monitoring` เดียวกัน สามารถติดต่อกันด้วยชื่อ container ได้
