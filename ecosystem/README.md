# Kiến trúc tổng thê
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

Ref: https://chatgpt.com/share/68f2096d-0c58-800a-adb2-c4753c771912
