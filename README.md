# KểCon — bản sao lưu (repo riêng tư)

> ⚠️ Repo này phải luôn để **Private**. Trong đó có truyện, tranh và giọng đọc của các gia đình.

Máy chủ staging tự sao lưu lên đây mỗi ngày lúc 03:15 (giờ Việt Nam) bằng `tools/kecon_backup.py`
(bản gốc: `deploy/prized/backup/` trong repo app; systemd `kecon-backup.timer`). Lần chạy gần nhất: **2026-10-16 11:40**.
Mỗi lần chạy đều giải mã bản chụp DB mới nhất và khôi phục thử vào một database tạm để chắc là dùng được.

| Phần | Ở đâu | Dạng |
| --- | --- | --- |
| Truyện (40 truyện, 0 truyện gia đình) | [`stories/`](stories/README.md) | đọc được: `README.md` + `story.json` (mọi cột) |
| Tranh + audio của truyện (603 file, 107 MB) | `media/<bucket>/<đường dẫn trong Storage>` | file gốc, `media/manifest.json` ghi sha256 |
| Ghi âm giọng bố mẹ (0 file) | `encrypted/recordings.tar.gpg` | **mã hoá** AES-256 |
| Cấu hình máy chủ (.env, secrets, systemd) | `encrypted/server-config.tar.gpg` | **mã hoá** AES-256 |
| Toàn bộ cơ sở dữ liệu (tài khoản, hộ, nhiệm vụ, …) | nhánh [`db-snapshots`](../../tree/db-snapshots) | **mã hoá** AES-256, 7 ngày + 4 tuần + 6 tháng |

## Giải mã

Cần **mật khẩu sao lưu** (bố mẹ/chủ dự án giữ; trên máy chủ ở `/srv/kecon/backup-github/passphrase`).
Không có mật khẩu thì phần mã hoá **không thể** khôi phục — phần truyện/tranh/audio vẫn đọc được bình thường.

```bash
gpg --decrypt encrypted/recordings.tar.gpg > recordings.tar      # hỏi mật khẩu
gpg --decrypt db-YYYYMMDDTHHMMSSZ.dump.gpg > kecon.dump          # file ở nhánh db-snapshots
```

## Khôi phục cơ sở dữ liệu

```bash
pg_restore -l kecon.dump | head                                   # xem mục lục
createdb -U postgres kecon_restore
pg_restore -U postgres -d kecon_restore --no-owner --no-privileges kecon.dump
```

Dump lấy bằng `pg_dump -Fc` của database `postgres` trong Supabase (gồm cả schema `auth`, `storage`, `public`).
Nên khôi phục vào một Supabase cùng phiên bản (Postgres 17.11) rồi chép lại `media/` vào Storage đúng bucket + đường dẫn.

## Khôi phục file Storage

`media/illustrations/…` và `media/tts-cache/…` giữ nguyên tên object; tải lại vào đúng bucket cùng tên.
Ghi âm: giải mã `recordings.tar` → `recordings/<đường dẫn>` → bucket `recordings`.
