# OpenClaw Restore Playbook (Host + Docker)

Mục tiêu: khôi phục toàn bộ OpenClaw khi mất container hoặc mất môi trường chạy.

Repo backup: `https://github.com/phongzou22022006-stack/backup_openclaw`

---

## 1) Chuẩn bị

Bạn cần có:
- repo backup chứa file `.enc`
- passphrase giải mã
- công cụ: `git`, `openssl`, `tar`, `rsync`
- `openclaw` (để verify backup)

Cài nhanh trên Debian/Ubuntu:

```bash
sudo apt update
sudo apt install -y git openssl tar rsync
```

---

## 2) Quy trình chung (áp dụng cho mọi môi trường)

### Bước 1: Clone repo backup

```bash
git clone https://github.com/phongzou22022006-stack/backup_openclaw.git
cd backup_openclaw/backups
ls -lah
```

### Bước 2: Giải mã file backup

Đổi tên file nếu backup mới hơn.

```bash
openssl enc -d -aes-256-cbc -pbkdf2 \
  -in openclaw-backup-20260327-195857.enc \
  -out openclaw-backup.tar.gz \
  -pass pass:'Fong@2006'
```

### Bước 3: Verify backup

```bash
openclaw backup verify openclaw-backup.tar.gz
```

### Bước 4: Giải nén

```bash
mkdir -p /tmp/restore-openclaw
tar -xzf openclaw-backup.tar.gz -C /tmp/restore-openclaw
```

Backup hiện tại chứa payload theo dạng:

`*/payload/posix/root/.openclaw/`

---

## 3) Scenario A — OpenClaw chạy trực tiếp trên host

### Khôi phục state

```bash
mkdir -p /root/.openclaw
rsync -a /tmp/restore-openclaw/*/payload/posix/root/.openclaw/ /root/.openclaw/
```

### Khởi động lại dịch vụ

```bash
openclaw gateway start
openclaw status
```

---

## 4) Scenario B — OpenClaw chạy trong Docker

### Nguyên tắc

Không lưu state chỉ trong container ephemeral.
Luôn mount host path vào `/root/.openclaw`.

Ví dụ host path: `/opt/openclaw-data`

### Bước 1: dừng/xóa container cũ (nếu có)

```bash
docker stop openclaw || true
docker rm openclaw || true
```

### Bước 2: chuẩn bị data dir trên host

```bash
sudo mkdir -p /opt/openclaw-data
sudo chown -R $USER:$USER /opt/openclaw-data
```

### Bước 3: restore backup vào host volume path

```bash
rsync -a /tmp/restore-openclaw/*/payload/posix/root/.openclaw/ /opt/openclaw-data/
```

### Bước 4: chạy lại container với volume mount

```bash
docker run -d \
  --name openclaw \
  -v /opt/openclaw-data:/root/.openclaw \
  -p 18789:18789 \
  <openclaw-image>
```

### Bước 5: kiểm tra

```bash
docker logs -f openclaw
# hoặc
docker exec -it openclaw sh -lc 'openclaw status'
```

---

## 5) Docker Compose mẫu

```yaml
services:
  openclaw:
    image: <openclaw-image>
    container_name: openclaw
    ports:
      - "18789:18789"
    volumes:
      - /opt/openclaw-data:/root/.openclaw
    restart: unless-stopped
```

Khởi động:

```bash
docker compose up -d
docker compose logs -f openclaw
```

---

## 6) Sự cố thường gặp

1. Restore xong vẫn mất dữ liệu:
- Mount sai path, hoặc restore sai path.
- Phải chắc chắn runtime dùng đúng `/root/.openclaw` đã restore.

2. Quyền file lỗi:
- Sửa quyền/chủ sở hữu data dir host.

3. Không giải mã được `.enc`:
- Sai passphrase.

4. `host.docker.internal` không hoạt động trên Linux Docker:
- Thêm host-gateway mapping khi cần:

```bash
--add-host=host.docker.internal:host-gateway
```

---

## 7) Checklist nhanh

- [ ] Clone repo backup thành công
- [ ] Giải mã `.enc` thành công
- [ ] `openclaw backup verify` pass
- [ ] Restore vào đúng path state
- [ ] Runtime (host hoặc docker) mount đúng state path
- [ ] `openclaw status` hoạt động lại

---

## 8) Bảo mật

- Token GitHub/PAT nếu từng lộ trong chat/log: revoke và tạo token mới.
- Không commit file backup chưa mã hóa lên remote.
- Passphrase càng mạnh càng tốt (bản hiện tại đang yếu, nên thay ở vòng backup sau).
