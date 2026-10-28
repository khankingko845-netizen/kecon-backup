# KểCon — bản sao lưu (repo riêng tư)

> ⚠️ Repo này phải luôn để **Private**. Trong đó có truyện, tranh và giọng đọc của các gia đình.

Máy chủ staging tự sao lưu lên đây mỗi ngày lúc 03:15 (giờ Việt Nam; đổi giờ / số bản giữ lại / token ở
**Admin › Sao lưu**) bằng `tools/kecon_backup.py` (bản gốc: `deploy/prized/backup/` trong repo app; systemd
`kecon-backup.timer`). Lần chạy gần nhất: **2026-10-29 03:16**.
Mỗi lần chạy đều giải mã bản chụp DB mới nhất và **khôi phục thử như khi chuyển máy** (user `supabase_admin`, giữ
owner + quyền, vào database trống), so số dòng và giải mã lại Vault (key nhà cung cấp AI) với DB thật; chỉ cần một
lỗi `pg_restore` là báo hỏng (`manifest.json` → `restore_check`).

| Phần | Ở đâu | Dạng |
| --- | --- | --- |
| Truyện (40 truyện, 0 truyện gia đình) | [`stories/`](stories/README.md) | đọc được: `README.md` + `story.json` (mọi cột) |
| Tranh + audio của truyện (683 file, 141 MB) | `media/<bucket>/<đường dẫn trong Storage>` | file gốc, `media/manifest.json` ghi version + sha256 |
| Bucket riêng tư (ghi âm bố mẹ, ảnh đại diện gia đình…) | `encrypted/family-avatars.tar.gpg` (0 file), `encrypted/recordings.tar.gpg` (0 file), `encrypted/voice-consents.tar.gpg` (0 file) | **mã hoá** AES-256 |
| Cấu hình máy chủ (`/srv/kecon/shared`: .env, secrets, **khoá gốc Vault**; systemd; `config.toml`) | `encrypted/server-config.tar.gpg` | **mã hoá** AES-256 |
| Toàn bộ cơ sở dữ liệu (tài khoản, hộ, nhiệm vụ, Vault, …) | nhánh [`db-snapshots`](../../tree/db-snapshots) | **mã hoá** AES-256, 7 ngày + 4 tuần + 6 tháng |
| Công cụ | `tools/kecon_backup.py`, `tools/kecon_restore.py` | Python 3, chỉ cần Docker |

## Giải mã

Cần **mật khẩu sao lưu** (bố mẹ/chủ dự án giữ; trên máy chủ ở `/srv/kecon/backup-github/passphrase`).
Không có mật khẩu thì phần mã hoá **không thể** khôi phục — phần truyện/tranh/audio vẫn đọc được bình thường.

```bash
gpg --decrypt encrypted/server-config.tar.gpg > server-config.tar   # hỏi mật khẩu
gpg --decrypt encrypted/recordings.tar.gpg > recordings.tar           # mỗi bucket riêng tư một file
gpg --decrypt db-YYYYMMDDTHHMMSSZ.dump.gpg > kecon.dump               # file ở nhánh db-snapshots
```

## Chuyển sang máy mới (khôi phục toàn bộ)

Dump là `pg_dump -Fc` của database `postgres` trong Supabase (cả schema `auth`, `storage`, `vault`, `public`).
Khôi phục vào stack Supabase CLI **cùng phiên bản** (Postgres 17.11; image DB `public.ecr.aws/supabase/postgres:17.11.0.004`, Storage
`public.ecr.aws/supabase/storage-api:v1.82.1` — `manifest.json` → `images`). Hai điều dễ sai: (1) phải khôi phục bằng **`supabase_admin`** —
bằng `postgres` thì Vault bị từ chối (`permission denied for table secrets`) và mất key AI; (2) stack mới phải dùng
**cùng khoá gốc Vault** (`srv/kecon/shared/pgsodium_root.key` trong server-config) — khoá khác thì Vault còn dòng
nhưng không giải mã được. File Storage phải nằm đúng `<bucket>/<tên>/<version>` và có content-type, nếu không
Storage trả `application/octet-stream`; `tools/kecon_restore.py` lo cả hai việc.

```bash
# 0. Lấy bản sao lưu + giải mã (mục trên): server-config.tar, <bucket>.tar, kecon.dump (bản mới nhất ở db-snapshots)
umask 077   # file giải mã chỉ user hiện tại đọc được
git clone https://github.com/khankingko845-netizen/kecon-backup.git && cd kecon-backup
git fetch origin db-snapshots && git show origin/db-snapshots:db-YYYYMMDDTHHMMSSZ.dump.gpg | gpg --decrypt > kecon.dump
# 1. Máy mới: sudo bash deploy/prized/setup-server.sh (repo app), đăng nhập lại
# 2. TRƯỚC bootstrap — cấu hình cũ: cùng jwt secret + khoá API (phiên đăng nhập giữ nguyên), key AI, khoá gốc Vault
mkdir -p /tmp/kc-config && tar -xf server-config.tar -C /tmp/kc-config
install -d -m 700 /srv/kecon/shared && install -m 600 /tmp/kc-config/srv/kecon/shared/* /srv/kecon/shared/
rm -f /srv/kecon/shared/public-url /srv/kecon/shared/release   # URL tunnel + release của máy mới tự sinh
# 3. bootstrap.sh như máy mới (deploy/prized/README.md) → stack Supabase tạo với khoá gốc cũ
# 4. Dừng app, thay DB, chép file Storage
sudo systemctl stop kecon-web kecon-backup.timer kecon-parent-notify.timer
python3 tools/kecon_restore.py db kecon.dump --yes --root-key /srv/kecon/shared/pgsodium_root.key
python3 tools/kecon_restore.py media . --private recordings.tar   # thêm --private <bucket>.tar cho mỗi bucket riêng tư
# 5. Migration mới hơn bản sao lưu (nếu có), bật lại app + timer
bash /srv/kecon/current/deploy/prized/deploy.sh /tmp/kecon-<sha>.tar.gz <sha>
```

`kecon_restore.py db` dừng các container khác của stack, `DROP DATABASE postgres WITH (FORCE)`, `pg_restore --create`
bằng `supabase_admin`, bật lại container rồi in số dòng + số secret Vault giải mã được (exit 4 nếu có lỗi). Nó từ
chối nếu DB đang có tài khoản (chống chạy nhầm trên máy đang dùng) trừ khi thêm `--replace-existing`, và dừng ngay
nếu `--root-key` khác khoá stack đang dùng. Xong thì đăng nhập bằng một tài khoản cũ và nghe thử một truyện.
Unit systemd + `config.toml` trong server-config chỉ để đối chiếu — `install-units.sh`/`supabase-stack.sh` tự tạo lại.
