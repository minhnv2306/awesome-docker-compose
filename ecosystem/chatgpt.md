# 1. Tôi là lập trình viên. Hướng dẫn tôi push log của Laravel, fie laravel.log lên Kibana

Rất hay — đây là một bước quan trọng khi bạn muốn **giám sát log Laravel** tập trung bằng **Kibana (Elastic Stack)**.
Tôi sẽ hướng dẫn bạn chi tiết **từng bước** để đẩy log từ Laravel (`storage/logs/laravel.log`) lên **Kibana**, thông qua **Filebeat + Elasticsearch** — cách phổ biến và chuẩn nhất trong thực tế DevOps.

---

## 🧩 Tổng quan luồng dữ liệu

```
Laravel (laravel.log)
   ↓
Filebeat (agent đọc log)
   ↓
Elasticsearch (lưu trữ & phân tích log)
   ↓
Kibana (trình hiển thị dashboard)
```

---

## 🚀 Cách thực hiện chi tiết

### **1. Bật log trong Laravel**

Laravel mặc định log tại:
`storage/logs/laravel.log`

Kiểm tra file `.env`:

```bash
LOG_CHANNEL=stack
LOG_LEVEL=debug
```

Tùy chọn format log để dễ parse hơn (trong `config/logging.php`):

```php
'channels' => [
    'stack' => [
        'driver' => 'stack',
        'channels' => ['single'],
        'ignore_exceptions' => false,
    ],

    'single' => [
        'driver' => 'single',
        'path' => storage_path('logs/laravel.log'),
        'level' => 'debug',
    ],
],
```

> ✅ Sau khi cấu hình, Laravel sẽ ghi log tại `storage/logs/laravel.log`.

---

### **2. Cài Filebeat trên server chứa Laravel**

**Ubuntu/Debian:**

```bash
sudo apt update
sudo apt install filebeat
```

**CentOS/RHEL:**

```bash
sudo yum install filebeat
```

---

### **3. Cấu hình Filebeat đọc file log Laravel**

Mở file cấu hình:

```bash
sudo nano /etc/filebeat/filebeat.yml
```

Tìm phần `filebeat.inputs:`
và thêm input cho Laravel:

```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /path/to/your/laravel/storage/logs/laravel.log
    fields:
      app: laravel
    fields_under_root: true
```

> ⚠️ Thay `/path/to/your/laravel` bằng đường dẫn thật trên server (ví dụ `/var/www/html/myapp`)

---

### **4. Gửi log đến Elasticsearch**

Trong cùng file `filebeat.yml`, chỉnh phần output:

```yaml
output.elasticsearch:
  hosts: ["http://localhost:9200"]
  username: "elastic"
  password: "your_password"
  index: "laravel-logs-%{+yyyy.MM.dd}"
```

> Nếu Elasticsearch nằm trên máy khác, thay `localhost` bằng IP hoặc domain của Elasticsearch server.

---

### **5. (Tuỳ chọn) Gửi qua Logstash**

Nếu bạn dùng **Logstash để xử lý log trước khi vào Elasticsearch**, thay phần output:

```yaml
output.logstash:
  hosts: ["logstash-server:5044"]
```

Và trong Logstash pipeline (`/etc/logstash/conf.d/laravel.conf`):

```conf
input {
  beats {
    port => 5044
  }
}

filter {
  grok {
    match => { "message" => "\[%{TIMESTAMP_ISO8601:timestamp}\] %{WORD:level}\.%{WORD:env}: %{GREEDYDATA:log_message}" }
  }
  date {
    match => ["timestamp", "yyyy-MM-dd HH:mm:ss"]
    timezone => "Asia/Ho_Chi_Minh"
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "laravel-logs-%{+YYYY.MM.dd}"
  }
}
```

---

### **6. Khởi động Filebeat**

```bash
sudo systemctl enable filebeat
sudo systemctl start filebeat
```

Kiểm tra log:

```bash
sudo tail -f /var/log/filebeat/filebeat
```

---

### **7. Xem log trên Kibana**

1. Mở Kibana:
   → `http://<kibana_host>:5601`

2. Vào **Discover → Create data view**
   → Nhập index pattern: `laravel-logs-*`

3. Bây giờ bạn sẽ thấy log từ Laravel hiển thị real-time 🎉

---

## 🧠 Mẹo mở rộng

* Nếu muốn log theo **JSON** để dễ phân tích:

  ```php
  'channels' => [
      'stack' => [
          'driver' => 'stack',
          'channels' => ['daily_json'],
      ],
      'daily_json' => [
          'driver' => 'daily',
          'path' => storage_path('logs/laravel.json'),
          'level' => 'debug',
          'formatter' => Monolog\Formatter\JsonFormatter::class,
      ],
  ]
  ```

  → Filebeat và Elasticsearch sẽ parse JSON dễ hơn.

---

## ✅ Tóm tắt nhanh

| Thành phần              | Nhiệm vụ                 |
| ----------------------- | ------------------------ |
| **Laravel**             | Sinh log                 |
| **Filebeat**            | Thu thập log từ file     |
| **Logstash (tùy chọn)** | Parse / chuyển đổi log   |
| **Elasticsearch**       | Lưu trữ log              |
| **Kibana**              | Hiển thị và truy vấn log |

---

Bạn có muốn tôi giúp bạn **viết luôn file cấu hình mẫu hoàn chỉnh (`filebeat.yml` + `logstash.conf`)** phù hợp với server Laravel của bạn (VD: Ubuntu, đường dẫn `/var/www/html/myapp`)?
Nếu có, bạn gửi cho tôi:

* OS server của bạn (Ubuntu/CentOS)
* Đường dẫn dự án Laravel
* Bạn có dùng Logstash trung gian hay gửi thẳng Elasticsearch?

# 2. Chi tiết cấu hình?
Tốt lắm 💪 — mình sẽ viết giúp bạn **bộ config hoàn chỉnh** để đẩy log Laravel (`laravel.log`) lên **Kibana**, gồm cả:

* `filebeat.yml`
* `logstash.conf`
  Cấu hình này áp dụng cho môi trường **Ubuntu**, Laravel nằm ở `/var/www/html/myapp`, và bạn **dùng Logstash** làm trung gian (chuẩn cho môi trường production).

---

## 🧩 Kiến trúc tổng thể

```
Laravel (laravel.log)
   ↓
Filebeat (agent đọc log)
   ↓
Logstash (parse & chuẩn hóa)
   ↓
Elasticsearch
   ↓
Kibana (xem log)
```

---

## ⚙️ 1️⃣ filebeat.yml (đặt tại `/etc/filebeat/filebeat.yml`)

```yaml
# ====================== Filebeat Inputs ======================
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/www/html/myapp/storage/logs/laravel.log
    fields:
      app: laravel
      env: production
    fields_under_root: true
    multiline.pattern: '^\['
    multiline.negate: true
    multiline.match: after

# ====================== Filebeat Modules ======================
# (Tắt tất cả module mặc định)
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false

# ====================== Elasticsearch output (tắt) ======================
#output.elasticsearch:
#  hosts: ["localhost:9200"]

# ====================== Logstash output ======================
output.logstash:
  hosts: ["localhost:5044"]

# ====================== Logging ======================
logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0644
```

> 💡 Giải thích nhanh:
>
> * `paths`: trỏ đến file `laravel.log`
> * `multiline.*`: gom các dòng lỗi nhiều dòng (trace) vào một bản ghi log duy nhất
> * `fields`: thêm metadata vào log (ví dụ app name)
> * `output.logstash`: gửi log qua Logstash

---

## ⚙️ 2️⃣ logstash.conf (đặt tại `/etc/logstash/conf.d/laravel.conf`)

```conf
input {
  beats {
    port => 5044
  }
}

filter {
  # Dạng log mặc định của Laravel:
  # [2025-10-17 12:34:56] production.ERROR: Message here {"context": "value"} []
  grok {
    match => {
      "message" => "\[%{TIMESTAMP_ISO8601:timestamp}\] %{WORD:env}\.%{LOGLEVEL:level}: %{GREEDYDATA:log_message}"
    }
  }

  # Cố gắng parse JSON nếu có
  json {
    source => "log_message"
    target => "json_data"
    skip_on_invalid_json => true
  }

  # Chuyển timestamp về dạng date chuẩn
  date {
    match => ["timestamp", "yyyy-MM-dd HH:mm:ss"]
    timezone => "Asia/Ho_Chi_Minh"
  }

  mutate {
    remove_field => ["host", "agent", "ecs", "input", "log", "timestamp"]
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "laravel-logs-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "your_password"
  }

  # Logstash console debug (tùy chọn)
  stdout { codec => rubydebug }
}
```

> 💡 Giải thích:
>
> * `input.beats`: nhận dữ liệu từ Filebeat qua port 5044
> * `grok`: tách thời gian, môi trường, mức độ log và nội dung log
> * `json`: nếu log có dạng JSON, tự động parse
> * `date`: chuyển chuỗi thời gian thành timestamp để Kibana đọc được
> * `output.elasticsearch`: đẩy dữ liệu vào Elasticsearch theo index mỗi ngày

---

## ⚙️ 3️⃣ Elasticsearch & Kibana

* Elasticsearch: chạy ở `localhost:9200`
* Kibana: chạy ở `localhost:5601`

Vào **Kibana → Discover → Create Data View**

→ Pattern: `laravel-logs-*`
→ Time field: `@timestamp`

Bạn sẽ thấy log Laravel hiển thị ngay 🎉

---

## ⚙️ 4️⃣ Khởi động dịch vụ

```bash
sudo systemctl enable logstash
sudo systemctl start logstash

sudo systemctl enable filebeat
sudo systemctl start filebeat
```

Kiểm tra log:

```bash
sudo tail -f /var/log/filebeat/filebeat
sudo tail -f /var/log/logstash/logstash-plain.log
```

---

## ✅ 5️⃣ Tùy chọn: Dùng log JSON cho Laravel (giúp parse dễ hơn)

Nếu bạn muốn log Laravel theo JSON (tốt hơn khi gửi qua Elasticsearch):

Trong `config/logging.php`:

```php
'channels' => [
    'stack' => [
        'driver' => 'stack',
        'channels' => ['daily_json'],
    ],

    'daily_json' => [
        'driver' => 'daily',
        'path' => storage_path('logs/laravel.json'),
        'level' => 'debug',
        'formatter' => Monolog\Formatter\JsonFormatter::class,
    ],
],
```

Rồi sửa `filebeat.yml` đọc file `laravel.json` thay vì `laravel.log`.

---

## 🧠 Kết quả

Sau khi hoàn tất:

* Kibana sẽ hiển thị đầy đủ các cột:

  * `@timestamp`
  * `level`
  * `log_message`
  * `env`
  * `app`
* Bạn có thể tạo dashboard thống kê lỗi, cảnh báo, hoặc filter theo `level:ERROR`.

---

Bạn muốn mình tạo luôn **dashboard mẫu trong Kibana** (gồm biểu đồ đếm lỗi theo thời gian, top 5 exception, và filter theo env)?
Mình có thể xuất file `.ndjson` để bạn import trực tiếp vào Kibana.
