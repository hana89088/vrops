# Hướng dẫn sử dụng vrops-exporter

Đây là tài liệu hướng dẫn cài đặt, cấu hình và sử dụng `vrops-exporter` để thu thập số liệu từ VMware vRealize Operations (vROps) và xuất chúng cho hệ thống giám sát Prometheus.

## Yêu cầu

- Python 3.6+ và `pip`.
- Quyền truy cập vào vROps API (tài khoản, mật khẩu).
- Máy chủ đã cài đặt Prometheus (để thu thập dữ liệu).

## 1. Cài đặt

Clone dự án về máy của bạn và cài đặt các thư viện Python cần thiết:

```bash
# Di chuyển vào thư mục dự án
cd vrops-exporter-master

# Cài đặt các thư viện từ file requirements.txt
pip install -r requirements.txt
```

## 2. Cấu hình

Exporter cần biết cách kết nối tới vROps và những dữ liệu nào cần thu thập. Bạn có thể cấu hình bằng file YAML.

Tạo một file tên là `config.yaml` trong thư mục gốc của dự án với nội dung sau và chỉnh sửa cho phù hợp với môi trường của bạn:

```yaml
# --- vROps Connection Settings ---
# Địa chỉ IP hoặc hostname của vROps
vrops_hostname: "your-vrops-hostname"
# Tên đăng nhập
vrops_user: "your-vrops-username"
# Mật khẩu
vrops_password: "your-vrops-password"
# Cổng API, mặc định là 443
vrops_port: 443
# Nguồn xác thực (ví dụ: Local, AD, LDAP)
vrops_auth_source: "Local" 

# --- Collector Settings ---
# Bật/tắt các collector tương ứng với loại dữ liệu bạn muốn thu thập.
# Tên collector phải khớp với tên class trong thư mục /collectors
collectors:
  # Thu thập số liệu hiệu năng của máy ảo (CPU, RAM,...)
  VMStatsCollector:
    enabled: true
  # Thu thập thuộc tính của máy ảo (tên, HĐH,...)
  VMPropertiesCollector:
    enabled: true
  # Thu thập số liệu của Host ESXi
  HostSystemStatsCollector:
    enabled: true
  # Thu thập thuộc tính của Host ESXi
  HostSystemPropertiesCollector:
    enabled: true
  # Thu thập số liệu của Cluster
  ClusterStatsCollector:
    enabled: true
  # Thu thập cảnh báo của Cluster
  ClusterAlertCollector:
    enabled: true
  # Các collector khác có thể được thêm vào đây
  # DatastoreStatsCollector:
  #   enabled: false
```

## 3. Các bước thực thi

Quá trình chạy gồm 2 bước chính: tạo cache và chạy exporter.

### Bước 1: Tạo Inventory Cache (`inventory.json`)

Để tránh phải truy vấn toàn bộ đối tượng trên vROps mỗi lần thu thập, chúng ta cần tạo một file cache chứa danh sách các đối tượng.

Chạy lệnh sau từ thư mục gốc. Lệnh này sẽ kết nối tới vROps, lấy danh sách tất cả đối tượng và lưu vào file `inventory.json`.

```bash
python inventory.py --config config.yaml
```
*Lưu ý: Lệnh này sẽ đọc thông tin kết nối từ `config.yaml` của bạn.*

### Bước 2: Chạy Exporter

Sau khi đã có file `inventory.json`, hãy khởi động exporter. Exporter sẽ đọc file inventory để biết cần lấy metric cho những đối tượng nào và đọc file `config.yaml` để biết collector nào đang được bật.

```bash
# Chạy exporter trên cổng 9101 (cổng mặc định không chính thức cho vrops-exporter)
# -i: Chỉ định file inventory cache
# -o: Chỉ định cổng để chạy web server
# --config: Chỉ định file cấu hình collector
python exporter.py -i inventory.json -o 9101 --config config.yaml
```

Nếu thành công, bạn sẽ thấy output tương tự như sau và tiến trình sẽ tiếp tục chạy:
```
[2025-11-05 17:30:00,123] [INFO] Starting exporter on port 9101
[2025-11-05 17:30:00,123] [INFO] Using inventory file: inventory.json
...
```

### Bước 3: Kiểm tra

Mở trình duyệt hoặc dùng `curl` để truy cập `http://localhost:9101/metrics`. Bạn sẽ thấy một danh sách dài các metrics được exporter xuất ra.

## 4. Tích hợp với Prometheus

Để Prometheus tự động thu thập dữ liệu, hãy thêm `job` sau vào file cấu hình `prometheus.yml` của bạn:

```yaml
scrape_configs:
  - job_name: 'vrops-exporter'
    static_configs:
      - targets: ['<IP_máy_chạy_exporter>:9101']
```

Thay `<IP_máy_chạy_exporter>` bằng địa chỉ IP của máy đang chạy tiến trình `exporter.py`. Khởi động lại Prometheus để áp dụng thay đổi.

## Xử lý sự cố thường gặp

- **Lỗi: `Cannot start, please specify port with ENV or -o`**
  - **Nguyên nhân:** Bạn chưa chỉ định cổng cho exporter.
  - **Giải pháp:** Thêm tham số `-o <port>` vào lệnh `python exporter.py`. Ví dụ: `-o 9101`.

- **Lỗi: `Cannot start, please specify inventory with ENV or -i`**
  - **Nguyên nhân:** Exporter không tìm thấy file inventory cache.
  - **Giải pháp:** Đảm bảo bạn đã chạy thành công `python inventory.py` ở Bước 1 và file `inventory.json` đã được tạo. Sau đó, thêm tham số `-i inventory.json` vào lệnh `python exporter.py`.
