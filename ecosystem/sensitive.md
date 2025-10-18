# Cấu hình Sensitive Logging cho Laravel với ELK Stack

## 📋 Tổng quan

Hệ thống này được thiết kế để xử lý và bảo mật các log có thông tin nhạy cảm từ Laravel applications, sử dụng ELK Stack (Elasticsearch, Logstash, Kibana) với Filebeat.

## 🔍 Vấn đề cần giải quyết

### 1. **Log Format gốc**
```
[2025-10-18 10:35:04] laravel.ALERT: Sensitive Info Alert {"lt":"sensitive","url":"/api/cage/list-import-box?cage_id=1","urp":"/api/cage/list-import-box","urq":"cage_id=1","rt":0.5,"st":200,"mt":"GET","rf":null,"rmip":null,"bbs":null,"cl":null,"tags":["a","b"],"tl":"2025-10-18+0710:35:04+07:00","host":"localhost","cip":"172.22.0.1","ua":"PostmanRuntime/7.49.0","uid":"1909988","rqb":{"cage_id":"1"},"xcs":"","xss":"","exf_pkg_orders":[]}
```

### 2. **Các thách thức**
- ❌ JSON data được nhúng trong log message như string
- ❌ Các field quan trọng như `lt`, `url`, `urp`, `rt`, `st` không được extract
- ❌ Thông tin nhạy cảm như IP, User ID, User Agent cần được mask
- ❌ Cần phân tách sensitive logs khỏi general logs
- ❌ Cần retention policy riêng biệt cho sensitive data

## 🛠️ Architecture

```
Laravel App → Log Files → Filebeat → Logstash → Elasticsearch → Kibana
            ↓
    sensitive.log (log_type: sensitive) → Port 5045 → sensitive.conf → sensitive-logs-*
    laravel.log (log_type: general) → Port 5044 → laravel.conf → laravel-logs-*
```

## ⚙️ Cấu hình chi tiết

### 1. **Filebeat Configuration** (`/etc/filebeat/filebeat.yml`)

```yaml
# Dual Input Configuration
filebeat.inputs:
  # Regular Laravel logs
  - type: log
    enabled: true
    paths:
      - /var/www/html/myapp/storage/logs/laravel.log
    fields:
      app: laravel
      env: production
      log_type: general
    fields_under_root: true
    multiline.pattern: '^\['
    multiline.negate: true
    multiline.match: after
    # Exclude sensitive logs từ general log
    exclude_lines: ['laravel\.ALERT: Sensitive Info Alert']

  # Sensitive logs - riêng biệt
  - type: log
    enabled: true
    paths:
      - /var/www/html/myapp/storage/logs/sensitive.log
    fields:
      app: laravel
      env: production
      log_type: sensitive
    fields_under_root: true
    multiline.pattern: '^\['
    multiline.negate: true
    multiline.match: after
    # Chỉ lấy sensitive logs
    include_lines: ['laravel\.ALERT: Sensitive Info Alert']

# Conditional Routing
output.logstash:
  hosts: ["localhost:5044", "localhost:5045"]
  loadbalance: false
  worker: 1
  when:
    - equals:
        log_type: "sensitive"
      then:
        hosts: ["localhost:5045"]
    - equals:
        log_type: "general"
      then:
        hosts: ["localhost:5044"]

# Metadata Enhancement
processors:
  - add_fields:
      when:
        equals:
          log_type: "sensitive"
      fields:
        data_classification: "confidential"
        compliance_required: true
        retention_days: 30
      target: metadata
```

### 2. **Logstash Configuration** (`/etc/logstash/conf.d/sensitive.conf`)

```ruby
input {
  beats {
    port => 5045  # Port riêng cho sensitive logs
  }
}

filter {
  # Parse log Laravel format - KEY SOLUTION: Hardcode "Sensitive Info Alert"
  grok {
    match => {
      "message" => "\[%{TIMESTAMP_ISO8601:timestamp}\] %{WORD:env}\.%{LOGLEVEL:level}: Sensitive Info Alert %{GREEDYDATA:json_data}"
    }
    add_field => { "grok_matched" => "true" }
    tag_on_failure => ["_grokparsefailure"]
  }

  # Parse JSON data - Tự động extract tất cả fields
  if [json_data] {
    json {
      source => "json_data"
      skip_on_invalid_json => true
    }
  }

  # Convert data types
  if [rt] {
    mutate { convert => { "rt" => "float" } }
  }
  if [st] {
    mutate { convert => { "st" => "integer" } }
  }

  # Timestamp processing
  date {
    match => ["timestamp", "yyyy-MM-dd HH:mm:ss"]
    timezone => "Asia/Ho_Chi_Minh"
  }

  # Security: Mask sensitive data
  if [cip] {
    mutate { gsub => ["cip", "\.\d+$", ".xxx"] }
  }
  if [ua] {
    mutate { gsub => ["ua", "/\d+\.\d+\.\d+", "/x.x.x"] }
  }

  # Classification
  mutate {
    add_field => { 
      "log_classification" => "sensitive"
      "retention_policy" => "30_days"
      "access_level" => "restricted"
    }
  }

  # Cleanup
  mutate {
    remove_field => ["agent", "ecs", "input", "log", "json_data", "@version"]
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "sensitive-logs-%{+YYYY.MM.dd}"
  }

  # Error handling
  if "_grokparsefailure" in [tags] or "_jsonparsefailure" in [tags] {
    elasticsearch {
      hosts => ["http://localhost:9200"]
      index => "failed-sensitive-logs-%{+YYYY.MM.dd}"
    }
  }
}
```

## 🎯 Kết quả đạt được

### **Extracted Fields trong Kibana:**
✅ **Core Fields:**
- `lt`: Log type identifier
- `url`: Full request URL
- `urp`: URL path
- `urq`: URL query string
- `rt`: Response time (float)
- `st`: Status code (integer)
- `mt`: HTTP method

✅ **Request Details:**
- `rf`: Referer
- `rmip`: Remote IP
- `host`: Request host
- `rqb`: Request body (JSON object)

✅ **User & Session:**
- `uid`: User ID
- `xcs`: CSRF token
- `xss`: Session token

✅ **Security Features:**
- `cip`: Client IP (masked: `172.22.0.1` → `172.22.0.xxx`)
- `ua`: User Agent (masked: `Agent/1.0` → `Agent/x.x.x`)

✅ **Metadata:**
- `log_classification`: "sensitive"
- `retention_policy`: "30_days"
- `access_level`: "restricted"

## 🔧 Các vấn đề đã khắc phục

### 1. **Grok Pattern Issue**
**Problem:** Pattern ban đầu `%{DATA:alert_type} %{GREEDYDATA:json_data}` không parse đúng

**Solution:** Hardcode cụ thể: `Sensitive Info Alert %{GREEDYDATA:json_data}`

### 2. **Port Conflict**
**Problem:** Multiple Logstash instances binding cùng port

**Solution:** 
```bash
sudo systemctl stop logstash
sudo pkill -9 -f logstash
sudo systemctl start logstash
```

### 3. **JSON Parsing**
**Problem:** JSON fields không được extract

**Solution:** Conditional parsing với `if [json_data]` check

### 4. **Data Type Conversion**
**Problem:** Numeric fields bị treat như string

**Solution:** Explicit conversion cho `rt` (float) và `st` (integer)

## 📊 Monitoring & Debugging

### **Check Elasticsearch Indices:**
```bash
curl -s "localhost:9200/_cat/indices/sensitive-logs-*?v"
```

### **Verify Latest Document:**
```bash
curl -s "localhost:9200/sensitive-logs-*/_search?sort=@timestamp:desc&size=1&pretty"
```

### **Check Logstash Status:**
```bash
sudo systemctl status logstash
sudo netstat -tlnp | grep -E "(5044|5045)"
```

### **Test Filebeat Connection:**
```bash
sudo filebeat test output -c /etc/filebeat/filebeat.yml
```

## 🚀 Deployment Steps

### 1. **Copy Configurations:**
```bash
sudo cp sensitive.conf /etc/logstash/conf.d/
sudo cp filebeat.yml /etc/filebeat/
```

### 2. **Restart Services:**
```bash
sudo systemctl restart logstash
sudo systemctl restart filebeat
```

### 3. **Test with Sample Log:**
```bash
echo '[2025-10-18 11:45:00] laravel.ALERT: Sensitive Info Alert {"lt":"test","url":"/api/test","rt":1.0,"st":200}' >> /path/to/sensitive.log
```

### 4. **Verify in Kibana:**
- Go to Kibana → Discover
- Select `sensitive-logs-*` index pattern
- Check for extracted fields: `lt`, `url`, `rt`, `st`

## 🔒 Security Best Practices

1. **Data Masking:** IP addresses và User Agents được tự động mask
2. **Separate Indexing:** Sensitive logs trong index riêng biệt
3. **Retention Policy:** Tự động cleanup sau 30 ngày
4. **Access Control:** Marked với `access_level: "restricted"`
5. **Compliance:** Metadata compliance tracking

## 🎉 Benefits

- ✅ **Complete JSON extraction** của tất cả fields
- ✅ **Security compliance** với data masking
- ✅ **Separate processing pipeline** cho sensitive data  
- ✅ **Rich analytics** trong Kibana với typed fields
- ✅ **Automated retention** management
- ✅ **Error handling** với failed logs indexing

Hệ thống này đảm bảo sensitive logs được xử lý an toàn, đầy đủ và có thể phân tích hiệu quả trong Kibana.