# Social Analyzer — Cài qua Portainer Git (mạng LAN)

Dùng fork [hoangthang0106/social-analyzer](https://github.com/hoangthang0106/social-analyzer), nhánh **`portainer-lan`**; không thay đổi Compose gốc.
**Cổng 18998/TCP là đề xuất chưa kiểm tra trên VM/LXC thật.** Backend nội bộ 9005/TCP không public.

## 1. Kiểm tra trước bằng lệnh chỉ đọc trên đúng VM/LXC Docker

```bash
hostname -I
docker ps -a --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
docker compose ls -a
sudo ss -lntup
sudo ss -ltnp '( sport = :18998 )'
docker ps -aq | xargs -r docker inspect --format '{{.Name}} {{json .HostConfig.PortBindings}}'
```

Không coi port trống nếu chưa xem listener và port được đặt cho container đang dừng. Nếu 18998 trùng, đặt `SOCIAL_ANALYZER_HOST_PORT` thành cổng TCP khác đã xác minh trống. Không dừng/xóa container cũ.

## 2. Tạo mật khẩu đã băm bằng Caddy

Trên Docker host, người quản trị **chủ động chạy** `docker run --rm -it caddy:2 caddy hash-password` và nhập mật khẩu khi chương trình hỏi (không ghi mật khẩu trong lệnh hoặc Git).
Sao chép chuỗi bcrypt (`$2a$` hoặc `$2b$`) để nhập vào Portainer. **OSINT_AUTH_HASH là hash, không phải mật khẩu thuần.**

## 3. Portainer → Stacks → Add stack → Repository

- **Stack name:** `osint-social-analyzer`
- **Repository URL:** `https://github.com/hoangthang0106/social-analyzer`
- **Reference:** `refs/heads/portainer-lan`
- **Compose path:** `portainer-compose.yml`
- **Environment variables** (nhập tay trước Deploy):
  - `LAN_IP`: IP thật trên interface của VM/LXC đang chạy Docker, không mặc định dùng IP Proxmox host.
  - `OSINT_AUTH_USER`: username dùng để mở web UI (ví dụ `itadmin`).
  - `OSINT_AUTH_HASH`: chuỗi bcrypt vừa tạo.
  - `SOCIAL_ANALYZER_HOST_PORT`: **tùy chọn**, chỉ nhập nếu cần đổi host port 18998.

**Có bước build image từ fork:** Repo dùng `build:` (context thư mục gốc). Portainer ở remote endpoint có thể **không hỗ trợ** build theo Compose. Nếu thất bại, không tự thay đổi service cũ; cần build image trên đúng VM/LXC bằng lệnh Docker Compose hoặc cấu hình Portainer local Docker endpoint, rồi Deploy lại. Hãy xem Logs trước khi xử lý. Dockerfile gốc dùng Node 18.15 trong khi package.json yêu cầu >=20.18.1. Dockerfile.portainer dùng Node 22, bỏ Selenium Grid/VNC của Compose gốc để tránh thêm port 4444/9002.

## 4. Kiểm tra sau khi Deploy (chỉ khi đã chủ động xác nhận)

- Đợi cả container ứng dụng và **gateway** Running, xem **Logs** nếu có lỗi.
- Mở **http://LAN_IP:18998** (thay port nếu đã đổi), nhập username và **mật khẩu gốc** bạn dùng tạo hash.
- Kiểm tra khả năng truy vấn dữ liệu bạn sở hữu hoặc được ủy quyền. Không có công cụ nào biết hết nền tảng mà một email/số điện thoại đã đăng ký.

## 5. Bảo mật & giới hạn

Gateway dùng Caddy `basic_auth`, backend không publish `ports:` ra host. Tuy nhiên **HTTP Basic Auth trên mạng LAN không mã hóa**; không mở cổng ra Internet, cần firewall giới hạn LAN; khi sử dụng dữ liệu nhạy cảm hãy chuyển sang HTTPS nội bộ. Người có quyền quản trị Portainer/Docker có thể xem biến môi trường. Không lưu khóa API, bí mật, dữ liệu nhạy cảm trong fork công khai.
Social Analyzer chủ yếu tìm username. Bản nhẹ không chạy Selenium Hub; một số chức năng webdriver/ảnh chụp sẽ hạn chế.
Dữ liệu ở Docker named volume (nếu có) không tự được backup. Không xóa volume hoặc bật tự cập nhật mà chưa kiểm tra. **Đây chỉ là cấu hình trên GitHub, không phải xác nhận máy chủ đã triển khai.**
