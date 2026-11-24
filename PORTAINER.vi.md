# Social Analyzer — Portainer Android — 18898:18898

**Chỉ cấu hình GitHub đã cập nhật. Không có bằng chứng port 18898/TCP thực tế trống hoặc container đã triển khai thành công trên VM/LXC Docker.**

## Công dụng

Tìm/so sánh dấu vết tài khoản mạng xã hội theo username; các kết quả không chứng minh cùng một chủ tài khoản.

## Cấu hình

- Fork: https://github.com/hoangthang0106/social-analyzer
- Git reference: `refs/heads/main`
- Compose path: **`docker-compose.yml`** (file mặc định đã đồng bộ với `portainer-compose.yml`)
- Stack name: `osint-social-analyzer`
- **Published Ports dự kiến:** `18898:18898/tcp` trên container gateway. Ứng dụng cũng nghe `18898` bên trong Docker network, nhưng **không publish trực tiếp**.
- Host port ở trên thuộc dãy 18894–18898, tiếp sau **18893 (Synaplan Lite)** do người dùng xác nhận là cổng đã sử dụng.
- Web server lấy biến PORT=18898, gateway nghe 18898. Dockerfile.portainer Node 22 dùng EXPOSE 18898.

## Bước 1 — Kiểm tra đúng máy Docker, lệnh chỉ đọc

```bash
hostname -I
ss -ltnp '( sport = :18898 )'
docker ps -a --format 'table {{.Names}}\t{{.Image}}\t{{.Ports}}'
docker ps -aq | xargs -r docker inspect --format '{{.Name}} {{json .HostConfig.PortBindings}}'
```

Kiểm tra listener và cả port được khai báo cho container đang dừng; không tự kết luận từ sổ GitHub. **Không dừng dịch vụ cũ để lấy port**. Nếu 18898 bị chiếm, không Deploy; kiểm kê rồi cấp cổng khác và sửa sổ port trước.

## Bước 2 — Mở Portainer bằng điện thoại

1. Chọn đúng **Environment / Docker endpoint**.
2. Vào **Stacks → Add stack → Repository** (hoặc cập nhật Stack đã tồn tại, không tạo trùng).
3. Repository URL: `https://github.com/hoangthang0106/social-analyzer.git`.
4. Reference: `refs/heads/main`. Compose path: `docker-compose.yml`.
5. Git authentication: `None` nếu repo công khai; tắt GitOps auto-update khi thử.
6. Phần **Environment variables**: nhập tay
   - `LAN_IP` = IP LAN thật của đúng VM/LXC có Docker (không đoán theo IP đã nhắc trong cuộc trò chuyện).
   - `OSINT_AUTH_USER` = tên đăng nhập bạn tự đặt.
   - `OSINT_AUTH_PASSWORD` = mật khẩu mạnh bạn tự đặt (KHÔNG cần hash).
   - `SOCIAL_ANALYZER_HOST_PORT` = tùy chọn, **không nhập** nếu dùng 18898. Nếu phải đổi port, vế bên trong gateway vẫn là 18898 và mapping không còn N:N; cần cập nhật sổ.
7. Chỉ sau khi kiểm tra port, volume và tài nguyên, mới **Deploy / Update the stack**.

Cần build từ mã nguồn fork; Compose mobile không chạy Selenium Hub/VNC để tránh các port phụ. Các chức năng cần trình duyệt tự động sẽ hạn chế. Giao diện tại /app.html.

## Bước 3 — Xác minh sau Deploy

- Portainer → Stacks → `osint-social-analyzer` → Containers: xem container ứng dụng và `gateway` có Running không.
- Published Ports dự kiến: gateway **`18898:18898`**; backend không publish cổng ngoài host.
- Truy cập: **`http://LAN_IP:18898/app.html`**.
- Nhập tên đăng nhập / mật khẩu tự đặt trong Environment variables.
- Nếu `reference not found`: kiểm tra URL Git, ref và Compose path. `port is already allocated`: kiểm tra đúng Docker host. `502 Bad Gateway`: đọc logs backend. Lỗi build: xem Dockerfile và Docker endpoint.

**Bảo mật:** Caddy Basic Auth qua HTTP LAN chưa mã hóa, đừng NAT ra Internet hoặc dùng mạng không tin cậy. Người có quyền quản trị Docker/Portainer có thể đọc biến môi trường. Đừng đưa mật khẩu, email, số điện thoại cá nhân hay API key lên GitHub công khai.

[Sổ quản lý port tập trung](https://github.com/hoangthang0106/thang/blob/main/ports/PORT_REGISTRY.vi.md). Cổng tiếp theo dự kiến là **18899**, chỉ cấp sau khi kiểm tra thực tế.
