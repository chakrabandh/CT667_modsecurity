# Nginx ModSecurity + HTML Frontend

โปรเจคนี้ประกอบด้วย Nginx พร้อม ModSecurity WAF เป็น reverse proxy และ HTML frontend แบบง่าย แยกเป็น 2 Docker Compose files

## โครงสร้างโปรเจค

```
project/
├── README.md
├── frontend/
│   ├── docker-compose-frontend.yml
│   └── index.html
└── modsecurity/
    ├── nginx/
    │   └── default.conf
    └── docker-compose-nginx.yml
```

## คุณสมบัติ

- **Nginx with ModSecurity**: WAF ที่มี OWASP Core Rule Set (CRS) v4
- **Reverse Proxy**: Nginx รับ request ที่ port 80 และ forward ไปยัง frontend
- **HTML Frontend**: Static HTML แสดง Hello World
- **Docker Network**: ใช้ shared network เพื่อให้ containers สื่อสารกันได้

## ข้อกำหนดเบื้องต้น

- Docker
- Docker Compose

## วิธีการติดตั้ง

### 1. รัน Nginx + ModSecurity

```bash
cd modsecurity
docker-compose -f docker-compose-nginx.yml up -d
cd ..
```

### 2. รัน Frontend

```bash
cd frontend
docker-compose -f docker-compose-frontend.yml up -d
cd ..
```

### 3. ทดสอบ

เปิดเบราว์เซอร์ไปที่ `http://localhost`

## คำสั่งที่ใช้บ่อย

### ดูสถานะ Containers

```bash
docker ps
```

### ดู Logs

```bash
# Nginx ModSecurity
docker logs nginx-modsecurity

# Frontend
docker logs html-frontend

# Follow logs แบบ real-time
docker logs -f nginx-modsecurity
```

### หยุดการทำงาน

```bash
docker-compose -f frontend/docker-compose-frontend.yml down
docker-compose -f modsecurity/docker-compose-nginx.yml down
```

### Restart Services

```bash
docker-compose -f modsecurity/docker-compose-nginx.yml restart
docker-compose -f frontend/docker-compose-frontend.yml restart
```

## การทดสอบ WAF (ModSecurity)

### 1. Request ปกติ

```bash
curl http://localhost
```

ควรได้ HTTP 200 พร้อม Hello World

### 2. ทดสอบ SQL Injection

```bash
curl "http://localhost/?id=1' OR '1'='1"
```

ควรได้ HTTP 403 Forbidden

### 3. ทดสอบ XSS

```bash
curl "http://localhost/?search=<script>alert('XSS')</script>"
```

ควรได้ HTTP 403 Forbidden

### 4. ทดสอบ Path Traversal

```bash
curl "http://localhost/../../../etc/passwd"
```

ควรได้ HTTP 403 Forbidden

### 5. ดู ModSecurity Audit Logs

```bash
docker exec -it nginx-modsecurity sh
tail -f /var/log/modsec_audit.log
```

## การตั้งค่า

### Nginx ModSecurity (modsecurity/docker-compose-nginx.yml)

- **Port**: 80 (external) → 8080 (internal)
- **ModSecurity Rules**: OWASP CRS v4
- **Paranoia Level**: 1 (ค่า default)
- **Anomaly Score Threshold**: Inbound=5, Outbound=4

### Frontend (frontend/docker-compose-frontend.yml)

- **Port**: 8080 (internal only, ไม่ expose ออกนอก)
- **Web Server**: Nginx Alpine

## Environment Variables (ModSecurity)

| Variable | ค่าที่ตั้ง | คำอธิบาย |
|----------|----------|----------|
| BACKEND | http://frontend | Backend service URL |
| PORT | 8080 | Nginx listening port |
| MODSEC_RULE_ENGINE | On | เปิดใช้งาน ModSecurity |
| MODSEC_REQ_BODY_ACCESS | On | ตรวจสอบ request body |
| MODSEC_REQ_BODY_LIMIT | 13107200 | ขนาด request body สูงสุด (12.5MB) |
| MODSEC_RESP_BODY_ACCESS | On | ตรวจสอบ response body |

## การปรับแต่ง

### เปลี่ยน Paranoia Level

แก้ไขใน `docker-compose-nginx.yml`:

```yaml
environment:
  - PARANOIA=2  # เพิ่มความเข้มงวด (1-4)
```

### เพิ่ม Custom Rules

สร้างไฟล์ `modsecurity/custom-rules.conf` และเพิ่ม volume:

```yaml
volumes:
  - ./custom-rules.conf:/etc/modsecurity.d/custom-rules.conf:ro
```

### แก้ไข HTML

แก้ไขไฟล์ `frontend/index.html` แล้ว restart:

```bash
docker-compose -f frontend/docker-compose-frontend.yml restart
```

## Troubleshooting

### Container ไม่ทำงาน

```bash
# ดู logs เพื่อหา error
docker logs nginx-modsecurity
docker logs html-frontend

# ตรวจสอบ network
docker network ls
docker network inspect app-network
```

### 403 Forbidden บน Request ปกติ

ModSecurity อาจจะ block request ที่ถูกต้อง ให้ตรวจสอบ logs:

```bash
docker logs nginx-modsecurity | grep -i "ModSecurity"
```

### Port Conflict

ถ้า port 80 ถูกใช้งานแล้ว แก้ไขใน `docker-compose-nginx.yml`:

```yaml
ports:
  - "8000:8080"  # เปลี่ยนจาก 80 เป็น 8000
```

## Security Notes

- ModSecurity CRS ช่วยป้องกัน OWASP Top 10 ได้หลายข้อ แต่ไม่ใช่ทั้งหมด
- ควรใช้ร่วมกับ secure coding practices
- ควร monitor logs เป็นประจำ
- พิจารณาเพิ่ม rate limiting และ DDoS protection

## License

MIT License

## ผู้พัฒนา

สร้างโดย Claude & User