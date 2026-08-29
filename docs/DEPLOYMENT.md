
- Nginx chạy trực tiếp trên VPS host (không container hóa) để đơn giản hóa việc quản lý chứng chỉ SSL và renewal.
- PostgreSQL/Redis chỉ giao tiếp qua Docker network nội bộ, không expose port ra Internet — giảm bề mặt tấn công.
- VPS: Ubuntu 24.04 LTS, 2 vCPU / 3GB RAM / 40GB NVMe.

## Quy trình deploy

Script `deploy.sh` tự động hóa toàn bộ quy trình cập nhật production:

```bash
./deploy.sh
```

Thực hiện tuần tự: `git pull` → rebuild image (`docker compose up -d --build`) → chờ healthcheck (tối đa 60s) → báo cáo kết quả kèm log nếu thất bại. Loại bỏ rủi ro quên bước hoặc deploy sai thứ tự khi thao tác thủ công.

## Bảo mật & hardening

- SSH: xác thực bằng key, tắt đăng nhập password, user riêng không dùng root cho ứng dụng.
- Firewall (UFW): chỉ mở port 22 (SSH), 80/443 (HTTP/HTTPS) — mọi service khác chỉ truy cập được qua network nội bộ Docker.
- Secrets: quản lý qua biến môi trường (`.env`), không commit vào Git; secret rotate ngay khi phát hiện rủi ro lộ.
- Reverse proxy là điểm truy cập duy nhất từ Internet — database/cache không bao giờ expose trực tiếp.

## Backup & Disaster Recovery

- `pg_dump` tự động hàng ngày (cron), nén gzip, giữ 7 bản gần nhất trên VPS, đồng bộ định kỳ ra ngoài VPS.
- **Đã kiểm chứng bằng restore test thực tế**: backup được restore vào database tạm, đối chiếu số bảng và số dòng dữ liệu khớp 100% với bản gốc — không chỉ tạo backup mà chưa từng xác nhận có dùng được hay không.

## Healthcheck & Monitoring

- Healthcheck Docker cho cả 4 service (frontend, backend, PostgreSQL, Redis) — backend kiểm tra cả kết nối database (`GET /health` qua `@nestjs/terminus`), không chỉ kiểm tra process còn sống.
- Uptime monitoring từ bên ngoài VPS (UptimeRobot) — phát hiện sự cố ngay cả khi toàn bộ VPS ngừng phản hồi, không phụ thuộc vào chính hệ thống đang được theo dõi.
- Log rotation (`max-size: 10m, max-file: 3` mỗi service) — tránh log tích lũy vô hạn làm đầy ổ đĩa.

## Performance & Load Testing

Kiểm chứng bằng [k6](https://k6.io/), giả lập 20 user đồng thời gọi các endpoint dashboard admin (nặng nhất về đọc dữ liệu — nhiều aggregate query):

|  Chỉ số   | Kết quả               |
|-----------|-----------------------|
| p95       | 318.68ms              |
| p99       | 341.09ms              |
| Tỷ lệ lỗi | 0% (0/2914 requests)  |

Kết luận: hạ tầng hiện tại (2 vCPU/3GB RAM) đủ dư địa cho quy mô vận hành thực tế (peak 50-100 user, concurrent 20-30 user theo mùa cao điểm).

## Database Migration

Migration chạy tự động khi container backend khởi động (`prisma migrate deploy` trong entrypoint script), đảm bảo schema production luôn đồng bộ với code deploy — không cần thao tác thủ công riêng.

## Stack vận hành

Containerization: Docker, Docker Compose 
Reverse proxy: Nginx 
SSL: Let's Encrypt / Certbot 
Process/Container health: Docker healthcheck 
Uptime monitoring: UptimeRobot 
Load testing: k6 
Backup: pg_dump + cron 