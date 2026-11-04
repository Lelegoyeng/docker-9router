# 9Router di Docker

Menjalankan [9Router](https://github.com/decolua/9router) (AI router/gateway) di dalam container Docker, dengan data persisten dan bisa diakses dari luar container.

## Isi repo

| File | Fungsi |
| --- | --- |
| `docker-compose.yml` | Definisi service 9Router, port, volume, healthcheck |
| `.env` | Konfigurasi & secret (tidak ikut ter-commit) |
| `.env.example` | Template konfigurasi |
| `.gitignore` | Mengecualikan `.env` dan file database |

## Prasyarat

- Docker Engine + Docker Compose v2 (`docker compose version`)
- Port `8001` bebas di host

## Menjalankan

```bash
# 1. Buat .env dari template (lewati jika .env sudah ada)
cp .env.example .env

# 2. Isi password & secret. Generate secret dengan:
openssl rand -hex 32

# 3. Jalankan
docker compose up -d

# 4. Cek status (tunggu sampai "healthy")
docker compose ps
```

Buka dashboard:

```
http://localhost:8001/dashboard
```

> Port yang diekspos ke luar adalah **8001** (`host:8001 -> container:20128`).
> Port internal container tetap `20128` dan tidak perlu diubah.

Password login pertama = nilai `INITIAL_PASSWORD` di `.env`.

> `INITIAL_PASSWORD` hanya dipakai saat database belum punya password tersimpan.
> Setelah login pertama, ganti password lewat dashboard. Mengubah `.env` setelah itu
> tidak mengubah password yang sudah tersimpan.

## Mengakses dari luar

Port sudah dipublish sebagai `0.0.0.0:8001` sehingga bisa diakses dari mesin lain
di jaringan yang sama:

```
http://<IP-host>:8001/dashboard
```

### Skenario A — VPS / server dengan IP publik

1. Buka firewall host:
   ```bash
   sudo ufw allow 8001/tcp
   ```
2. Buka juga port `8001` di security group/firewall cloud provider (AWS, GCP, Azure, dll).
3. Sesuaikan `.env`:
   ```env
   BASE_URL=http://<IP-publik>:8001
   NEXT_PUBLIC_BASE_URL=http://<IP-publik>:8001
   ```
4. Terapkan:
   ```bash
   docker compose up -d
   ```

Akses: `http://<IP-publik>:8001/dashboard`.

### Skenario B — GitHub Codespaces (environment ini)

Port sudah di-forward otomatis. URL publik saat ini:

```
https://effective-fortnight-74q9rpjrvj42rvrj-8001.app.github.dev/dashboard
```

Pastikan visibility port `public` (bukan private) agar bisa dibuka tanpa login GitHub:

```bash
gh codespace ports visibility 8001:public -c "$CODESPACE_NAME"
```

Untuk mengembalikan ke private:

```bash
gh codespace ports visibility 8001:private -c "$CODESPACE_NAME"
```

> URL Codespaces berubah jika codespace dibuat ulang. Perbarui `BASE_URL` di `.env`
> lalu jalankan `docker compose up -d` lagi.

### Skenario C — Domain + HTTPS (disarankan untuk produksi)

Jangan ekspos port `8001` langsung ke internet. Pakai reverse proxy (Caddy paling mudah
karena sertifikat HTTPS otomatis):

```caddyfile
router.contoh.com {
    reverse_proxy 127.0.0.1:8001
}
```

Lalu:

1. Ubah binding di `.env` agar hanya lokal:
   ```env
   ROUTER_BIND_ADDR=127.0.0.1
   ```
2. Set URL publik:
   ```env
   BASE_URL=https://router.contoh.com
   NEXT_PUBLIC_BASE_URL=https://router.contoh.com
   ```
3. Jalankan ulang:
   ```bash
   docker compose up -d
   ```

Contoh Nginx:

```nginx
server {
    listen 80;
    server_name router.contoh.com;

    location / {
        proxy_pass http://127.0.0.1:8001;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;          # penting untuk streaming SSE
        proxy_read_timeout 3600s;
    }
}
```

> Streaming respons AI butuh `proxy_buffering off` (Nginx) atau proxy yang meneruskan
> SSE apa adanya. Tanpa itu, respons bisa tertahan.

## Memakai 9Router dari CLI/AI coding tool

Ambil API key di dashboard (`Endpoint` / `API Keys`), lalu arahkan tool ke:

```
Base URL : http://<host-atau-domain>:8001/v1
API Key  : <key dari dashboard>
Model    : mis. kr/claude-sonnet-4.5 atau nama combo
```

Contoh Claude Code / tool kompatibel OpenAI:

```bash
export OPENAI_BASE_URL="https://router.contoh.com/v1"
export OPENAI_API_KEY="<key-dari-dashboard>"
```

## Menghubungkan OpenCode (manual)

Install OpenCode:

```bash
curl -fsSL https://opencode.ai/v2/install | bash
```

Buat `~/.config/opencode/opencode.json`:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "model": "9router/cc/claude-sonnet-5-5",
  "providers": {
    "9router": {
      "name": "9Router",
      "package": "@opencode/ai/providers/openai-compatible",
      "settings": {
        "baseURL": "http://localhost:8001/v1",
        "apiKey": "{file:~/.config/opencode/9router.key}"
      },
      "models": {
        "cc/claude-sonnet-5-5": { "name": "Claude Sonnet 5.5" },
        "cx/gpt-6.1-sol": { "name": "GPT-6.1 Sol" }
      }
    }
  }
}
```

Simpan API key dari dashboard ke `~/.config/opencode/9router.key`, lalu:

```bash
chmod 600 ~/.config/opencode/9router.key
```

Sesuaikan `baseURL` kalau 9Router tidak di `localhost:8001`, dan tambahkan model lain
sesuai yang tersedia. Daftar model bisa dilihat dengan:

```bash
curl -H "Authorization: Bearer <key>" http://localhost:8001/v1/models | jq -r '.data[].id'
```

Setelah itu jalankan `opencode` dan pilih model lewat `/models`.

> Menu **CLI Tools → OpenCode** di dashboard 9Router menulis config di dalam container,
> bukan di host. Karena OpenCode dijalankan di host, edit config manual seperti di atas.

## Data & backup

Data disimpan di volume Docker `9router-data` (di container: `/app/data`, database
SQLite di `/app/data/db/data.sqlite`).

Backup:

```bash
docker run --rm -v 9router-data:/data -v "$PWD":/backup alpine \
  tar czf /backup/9router-backup-$(date +%F).tar.gz -C /data .
```

Restore:

```bash
docker compose down
docker run --rm -v 9router-data:/data -v "$PWD":/backup alpine \
  sh -c 'rm -rf /data/* && tar xzf /backup/9router-backup-YYYY-MM-DD.tar.gz -C /data'
docker compose up -d
```

Reset total (hapus semua data, termasuk password & koneksi provider):

```bash
docker compose down -v
docker compose up -d
```

## Update 9Router

```bash
docker compose pull
docker compose up -d
```

Untuk mengunci versi tertentu, set di `.env`:

```env
ROUTER_IMAGE_TAG=0.5.81
```

## Operasional

```bash
docker compose logs -f          # lihat log
docker compose restart          # restart
docker compose stop             # stop
docker compose down             # hapus container (data tetap)
docker stats 9router            # pemakaian resource
```

Health check manual:

```bash
curl http://localhost:8001/api/health
# {"ok":true}
```

## Keamanan

- Ganti `INITIAL_PASSWORD` dan semua secret di `.env` sebelum deploy publik.
- Jangan commit `.env` (sudah masuk `.gitignore`).
- Untuk akses internet, utamakan reverse proxy HTTPS + `ROUTER_BIND_ADDR=127.0.0.1`.
- Dashboard menyimpan OAuth token/API key provider. Perlakukan akses ke dashboard
  seperti akses ke akun AI kamu.
- Membuka port Codespaces sebagai `public` berarti siapa pun yang tahu URL bisa
  mencapai halaman login. Gunakan password kuat dan set kembali ke `private` bila
  tidak dipakai.

## Troubleshooting

| Masalah | Solusi |
| --- | --- |
| `port is already allocated` | Ganti `ROUTER_PORT` di `.env`, mis. `20129`, lalu `docker compose up -d` |
| Status tidak `healthy` | `docker compose logs --tail 100`; pastikan data volume bisa ditulis |
| Dashboard 502 via reverse proxy | Pastikan proxy mengarah ke `127.0.0.1:8001` dan container hidup |
| Streaming terpotong | Matikan buffering di reverse proxy (`proxy_buffering off`) |
| Lupa password | Reset dengan `docker compose down -v` (menghapus semua data) |
