# UMKM Insight Assistant

### 🎓 Capstone Project & Certification

<table>
  <tr>
    <td align="center" width="50%">
      <a href="https://www.credly.com/badges/bfbc7eeb-4c6e-46e0-ba7e-ef8f049bac24">
        <img src="https://images.credly.com/size/340x340/images/d8f30e8e-4c24-42e8-bb15-b106bb082614/BadgeEmblem_BuildAnAIAgent.png" width="200"><br>
        <sub><b>IBM SkillsBuild Badge</b></sub>
      </a>
    </td>
    <td align="center" width="50%">
      <a href="/serifikat-dan-badges-ibm-skillsbuild/hactiv8-certificate-ibm-skilsbuild-1.png">
        <img src="/serifikat-dan-badges-ibm-skillsbuild/hactiv8-certificate-ibm-skilsbuild-1.png" width="220"><br>
        <sub><b>Hacktiv8 Certificate </b></sub>
      </a>
    </td>
  </tr>
</table>
---

AI Agent berbasis Langflow untuk membantu pemilik UMKM toko online (fashion & gadget) mendapatkan insight penjualan melalui percakapan bahasa natural — tanpa perlu menulis query SQL atau membuat pivot table manual.

## Latar Belakang

Banyak UMKM mencatat transaksi secara manual di spreadsheet, namun pemiliknya umumnya tidak memiliki latar belakang teknis untuk melakukan analisis data. UMKM Insight Assistant menjawab masalah ini dengan menyediakan antarmuka percakapan: pengguna cukup bertanya (misal *"Produk apa yang paling laku bulan ini?"*), dan sistem akan menerjemahkan pertanyaan ke query SQL, mengeksekusinya secara aman ke database, lalu merangkum hasilnya menjadi insight yang mudah dipahami.

## Arsitektur

```
Chat Input → Agent (Gemini) → SQL Database (Tool, read-only) → Agent → Chat Output
```

Agent menerima pertanyaan dalam bahasa natural, menyusun query SQL berdasarkan skema database yang dijelaskan di system prompt, mengeksekusi query melalui tool SQL Database (menggunakan user database read-only demi keamanan), lalu menyusun jawaban naratif dari hasil query.

## Prasyarat

- Docker & Docker Compose sudah terinstall
- API Key Google Gemini (gratis, daftar di https://aistudio.google.com/apikey)

## Struktur File

```
capstone-ai-agent/
├── docker-compose.yml
├── seed_data.sql
├── flow_export.json
└── README.md
```

## Langkah Setup

### 1. Jalankan Docker

```bash
docker compose up -d
```

Tunggu hingga kedua container (`langflow-postgres` dan `langflow`) berstatus "Up". Cek dengan:

```bash
docker compose ps
```

### 2. Load Seed Data ke Database

```bash
docker cp seed_data.sql langflow-postgres:/seed_data.sql
docker exec -it langflow-postgres psql -U langflow -c "CREATE DATABASE umkm_store;"
docker exec -it langflow-postgres psql -U langflow -d umkm_store -f /seed_data.sql
```

Verifikasi data berhasil masuk:

```bash
docker exec -it langflow-postgres psql -U langflow -d umkm_store -c "
SELECT
  (SELECT COUNT(*) FROM customers) as customers,
  (SELECT COUNT(*) FROM products) as products,
  (SELECT COUNT(*) FROM orders) as orders,
  (SELECT COUNT(*) FROM order_items) as order_items;"
```

Hasil yang diharapkan: `60 customers | 20 products | 670 orders | 1697 order_items`

### 3. Buat User Database Read-Only

Demi keamanan, Agent mengakses database menggunakan user khusus yang hanya bisa membaca data (tidak bisa `INSERT`/`UPDATE`/`DELETE`):

```bash
docker exec -it langflow-postgres psql -U langflow -d umkm_store -c "
CREATE USER agent_readonly WITH PASSWORD 'agentpass';
GRANT CONNECT ON DATABASE umkm_store TO agent_readonly;
GRANT USAGE ON SCHEMA public TO agent_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO agent_readonly;
"
```

### 4. Buka Langflow

Kunjungi http://localhost:7860 di browser.

### 5. Import Flow

Di halaman Projects, klik **Import**, lalu pilih file `flow_export.json`.

### 6. Set Global Variables (WAJIB sebelum flow bisa dijalankan)

Flow ini menggunakan Global Variables untuk kredensial, sehingga tidak ada API key/password yang tersimpan di dalam file JSON. Buat variable berikut:

Buka **profil (pojok kanan atas) → Settings → Global Variables → Add New**, lalu buat:

| Nama Variable | Value |
|---|---|
| `UMKM_DB_URL` | `postgresql://agent_readonly:agentpass@postgres:5432/umkm_store` |
| `GOOGLE_API_KEY` | *(API key Gemini masing-masing, dari https://aistudio.google.com/apikey)* |

### 7. Jalankan & Uji Coba

Buka flow yang sudah di-import, klik **Playground**, lalu coba pertanyaan berikut:

- *"Ada berapa total customer yang terdaftar?"* → jawaban seharusnya 60
- *"Produk apa yang paling laku bulan ini?"*
- *"Fashion atau gadget yang lebih banyak terjual?"*
- *"Bagaimana tren penjualan dari bulan ke bulan?"*

## Skema Database

**customers** — customer_id, name, phone, city, joined_date

**products** — product_id, product_name, category (Fashion/Gadget), brand, price, stock_qty

**orders** — order_id, customer_id, order_date, status (Completed/Shipped/Pending/Cancelled), payment_method, total_amount

**order_items** — item_id, order_id, product_id, quantity, subtotal

## Catatan Keamanan

- Agent mengakses database melalui user `agent_readonly` yang hanya memiliki hak `SELECT`, sehingga query berbahaya (misal hasil dari kesalahan generate atau prompt injection) akan ditolak database.
- Kredensial (Database URL dan API Key) disimpan sebagai Global Variables di Langflow, bukan hardcode di dalam flow, sehingga file `flow_export.json` aman untuk dibagikan tanpa membocorkan kredensial.

## Troubleshooting

**Flow macet lama di "Running..." tanpa respons:**
Kemungkinan bug tracing internal Langflow. Tambahkan environment variable berikut ke service `langflow` di `docker-compose.yml`, lalu jalankan ulang `docker compose up -d`:

```yaml
LANGFLOW_NATIVE_TRACING: "false"
```

**Error "permission denied for table" saat Agent query:**
Pastikan langkah 3 (grant akses read-only) sudah dijalankan dengan benar.

**Data hilang setelah restart:**
Pastikan tidak pernah menjalankan `docker compose down -v` (flag `-v` menghapus volume beserta seluruh datanya). Gunakan `docker compose down` biasa untuk mematikan tanpa menghapus data.
