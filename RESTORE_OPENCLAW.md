# Restore OpenClaw After Container Loss

Nếu container OpenClaw bị xoá nhưng bạn còn repo backup này, làm theo các bước sau.

## 1. Clone repo backup

```bash
git clone https://github.com/phongzou22022006-stack/backup_openclaw.git
cd backup_openclaw/backups
ls -lah
```

## 2. Giải mã file backup

Thay tên file `.enc` nếu backup mới hơn.

```bash
openssl enc -d -aes-256-cbc -pbkdf2 \
  -in openclaw-backup-20260327-195857.enc \
  -out openclaw-backup.tar.gz \
  -pass pass:'Fong@2006'
```

## 3. Verify archive

```bash
openclaw backup verify openclaw-backup.tar.gz
```

## 4. Giải nén backup

```bash
mkdir -p /tmp/restore-openclaw
tar -xzf openclaw-backup.tar.gz -C /tmp/restore-openclaw
```

## 5. Khôi phục state vào `~/.openclaw`

Backup hiện tại chứa payload dưới dạng:

- `*/payload/posix/root/.openclaw/`

Khôi phục bằng:

```bash
mkdir -p /root/.openclaw
rsync -a /tmp/restore-openclaw/*/payload/posix/root/.openclaw/ /root/.openclaw/
```

## 6. Chạy lại OpenClaw

```bash
openclaw gateway start
openclaw status
```

## Ghi chú

- Backup này là bản full-state, nên có cả config, sessions, workspace, memory, skills.
- Nếu chạy OpenClaw bằng Docker, nhớ mount lại đúng thư mục dữ liệu để state không biến mất lần nữa.
- Nếu token GitHub từng bị lộ trong chat/log, hãy revoke token cũ và tạo token mới.
