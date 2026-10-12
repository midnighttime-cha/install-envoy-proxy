# คู่มือการติดตั้งและตั้งค่า Envoy Proxy แบบ Modular (Zero-Downtime)

คู่มือฉบับนี้สรุปขั้นตอนตั้งแต่การติดตั้ง Envoy Proxy ไปจนถึงการตั้งค่าเพื่อรองรับหลายโดเมนและหลายระบบบนเซิร์ฟเวอร์ Ubuntu เดียวกัน โดยใช้เทคนิคแยกไฟล์ย่อย (`.d`) เพื่อให้จัดการง่ายและสามารถอัปเดตระบบได้โดยไม่ต้อง Restart Service (Zero-Downtime)

---

## 1. การติดตั้ง Envoy Proxy บน Ubuntu

อัปเดตระบบและติดตั้ง Envoy ผ่าน APT Repository:
```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gnupg2 software-properties-common
curl -sL 'https://deb.dl.getenvoy.io/public/gpg.8115BA8E629CC074.key' | sudo gpg --dearmor -o /usr/share/keyrings/getenvoy-keyring.gpg
echo a077cb587a1b622e03aa4bf2f3689de14658a9497a9af2c427bba5f4cc3c4723 | sudo tee /etc/apt/sources.list.d/getenvoy.list
sudo apt update
sudo apt install -y getenvoy-envoy
```

---

## 2. การเตรียมโครงสร้างโฟลเดอร์ (Directory Structure)

เราจะแยกไฟล์ย่อยของแต่ละระบบออกจากกัน โดยสร้างโฟลเดอร์สำหรับเก็บไฟล์ Clusters และ Routes:
```bash
sudo mkdir -p /etc/envoy/clusters
sudo mkdir -p /etc/envoy/routes
sudo mkdir -p /etc/envoy/clusters.d
sudo mkdir -p /etc/envoy/routes.d
```

---

## 3. การตั้งค่าไฟล์หลัก (`envoy.yaml`)

สร้างหรือแก้ไขไฟล์หลัก `/etc/envoy/envoy.yaml` เพื่อสั่งให้ Envoy ดึงข้อมูลแบบ Dynamic จากไฟล์ `proxy.yaml`:

```yaml
node:
  cluster: proxy_cluster
  id: proxy_node_1

dynamic_resources:
  cds_config:
    path_config_source:
      path: /etc/envoy/clusters/example-proxy.yaml # <---- แก้ชื่อไฟล์นี้ตามที่ตั้งค่า

static_resources:
  listeners:
  - name: listener_http
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 80
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          rds:
            config_source:
              path_config_source:
                path: /etc/envoy/routes/example-proxy.yaml # <---- แก้ชื่อไฟล์นี้ตามที่ตั้งค่า
            route_config_name: local_route
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
```
*(เมื่อแก้ไขไฟล์นี้เสร็จ ให้รัน `sudo systemctl restart envoy` เพื่อให้เริ่มทำงาน)*

---

## 4. สร้างสคริปต์รวมไฟล์อัตโนมัติ (`envoy-sync`)

สร้างสคริปต์สำหรับกวาดไฟล์ย่อยทั้งหมดมารวมเป็นไฟล์เดียว พร้อมระบบเคลียร์อักขระขยะ (Non-breaking space) อัตโนมัติ:

```bash
cat << 'EOF' | sudo tee /usr/local/bin/envoy-sync
#!/bin/bash
# 1. กวาดล้างอักขระพิเศษ (Non-breaking space) ในไฟล์ย่อยให้เป็น Spacebar ปกติ
sed -i 's/\xc2\xa0/ /g' /etc/envoy/clusters.d/*.yml 2>/dev/null || true
sed -i 's/\xc2\xa0/ /g' /etc/envoy/routes.d/*.yml 2>/dev/null || true

echo "กำลังรวมไฟล์ Cluster..."
echo "resources:" > /etc/envoy/clusters/example-proxy.yaml
cat /etc/envoy/clusters.d/*.yml >> /etc/envoy/clusters/example-proxy.yaml 2>/dev/null || true

echo "กำลังรวมไฟล์ Route..."
cat << 'ROUTE_HEADER' > /etc/envoy/routes/example-proxy.yaml
resources:
- "@type": type.googleapis.com/envoy.config.route.v3.RouteConfiguration
  name: local_route
  virtual_hosts:
ROUTE_HEADER
cat /etc/envoy/routes.d/*.yml >> /etc/envoy/routes/example-proxy.yaml 2>/dev/null || true

# 2. กวาดล้างในไฟล์หลักอีกรอบ
sed -i 's/\xc2\xa0/ /g' /etc/envoy/clusters/example-proxy.yaml
sed -i 's/\xc2\xa0/ /g' /etc/envoy/routes/example-proxy.yaml

echo "✅ อัปเดต Envoy เรียบร้อย (Zero-Downtime)!"
EOF

# กำหนดสิทธิ์ให้สคริปต์สามารถรันได้
sudo chmod +x /usr/local/bin/envoy-sync
```

---

## 5. วิธีเพิ่มระบบใหม่ (ตัวอย่าง: Gitea)

เมื่อต้องการเพิ่มระบบใหม่ ให้สร้างไฟล์ย่อย 2 ไฟล์ในโฟลเดอร์ `.d` ดังนี้:

**1. ไฟล์ Cluster (ตัวอย่าง: `/etc/envoy/clusters.d/example-app.yml`)**
*เริ่มที่ `- "@type":` ได้เลย ไม่ต้องมี resources:*
```yaml
- "@type": type.googleapis.com/envoy.config.cluster.v3.Cluster
  name: backend_example_app
  connect_timeout: 0.25s
  type: STRICT_DNS
  lb_policy: ROUND_ROBIN
  load_assignment:
    cluster_name: backend_example_app
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: 127.0.0.1
              port_value: 3000
```

**2. ไฟล์ Route (ตัวอย่าง: `/etc/envoy/routes.d/example-app.yml`)**
*สำคัญ: ต้องเคาะเว้นวรรค (Spacebar) 2 ครั้งในบรรทัดแรกเสมอ:*
```yaml
  - name: example_app_service
    domains: ["app.example-app.com"]
    routes:
    - match:
        prefix: "/"
      route:
        cluster: backend_example_app
```

---

## 6. การอัปเดตระบบแบบ Zero-Downtime

ทุกครั้งที่มีการ **เพิ่ม/ลบ/แก้ไข** ไฟล์ในโฟลเดอร์ `.d` ให้รันคำสั่งเพียงคำสั่งเดียว:
```bash
sudo envoy-sync
```
ระบบจะอัปเดตตั้งค่าทั้งหมดทันทีโดยไม่ทำให้เซิร์ฟเวอร์สะดุด

---

## 7. ปัญหาที่พบบ่อย (Troubleshooting)

1. **เจอ Error 404 Not Found (หลังจากรัน `envoy-sync` แล้ว):**
   * **สาเหตุ:** มักเกิดจากการเว้นวรรค (Indentation) ในไฟล์ `.yml` ย่อยผิดพลาด หรือลืมเคาะเว้นวรรค 2 ทีที่บรรทัดแรกของไฟล์ Route
   * **วิธีเช็ค:** รัน `sudo journalctl -u envoy -n 20` เพื่อดูว่า Envoy ปฏิเสธไฟล์ไหน

2. **เจอ Error 521 Web server is down (เมื่อใช้ผ่าน Cloudflare):**
   * **สาเหตุ:** Envoy ของเรารับทราฟฟิก HTTP (Port 80) แต่ Cloudflare พยายามเชื่อมต่อแบบ HTTPS (Port 443)
   * **วิธีแก้:** เข้าไปที่ระบบของ Cloudflare -> ไปที่เมนู SSL/TLS -> ปรับโหมดจาก Full (หรือ Strict) ให้เป็น **Flexible**

3. **เชื่อมต่อไม่ได้เลย (Connection Timeout):**
   * **สาเหตุ:** ถูก Firewall ของระบบบล็อกเอาไว้
   * **วิธีแก้:** ตรวจสอบ Security Group (หากใช้ Cloud Provider) หรือ UFW ในเครื่อง ว่าได้เปิดอนุญาต Inbound Port `80` และ `443` ให้เข้าถึงได้จาก `0.0.0.0/0` หรือยัง
