# PostgreSQL DBA Hands-On Practice: Dari Instalasi hingga Simulasi Skenario Production

**Roadmap lengkap 6 tahap latihan hands-on Database Administrator (PostgreSQL)**
Environment: Windows 11 + Docker Desktop (WSL2 backend) + PostgreSQL 16 + DBeaver + PgBouncer

---

## Ringkasan

Latihan ini dimulai dari nol — belum pernah install Docker, belum pernah setup PostgreSQL di luar konteks belajar SQL dasar. Enam tahap berikut membangun kompetensi DBA secara bertahap: **administrasi server & autentikasi** (Tahap 1), **access control berbasis role** (Tahap 2), **backup & disaster recovery** (Tahap 3), **performance tuning & monitoring** (Tahap 4), **high availability & replikasi** (Tahap 5), dan **simulasi skenario production** yang menggabungkan semuanya (Tahap 6).

Yang membuat latihan ini lebih dari sekadar mengikuti tutorial adalah **sembilan masalah nyata** yang muncul di sepanjang perjalanan — port conflict, error autentikasi lintas client-server, privilege dan objek sequence yang terpisah dari tabelnya, error restore akibat objek yang sudah ada, DNS resolution antar-container, permission ownership pada proses replikasi, hingga ketidakcocokan metode autentikasi pada connection pooler — dan proses investigasi sistematis untuk menyelesaikannya satu per satu, persis seperti alur kerja DBA sehari-hari.

---

## TAHAP 1 — Instalasi, Konfigurasi Server, dan Autentikasi

### Yang Dikerjakan

**1. Environment Setup**
- Install Docker Desktop di Windows dengan WSL2 backend
- Menjalankan PostgreSQL 16 sebagai container terisolasi:
  ```bash
  docker run --name pg-dba -e POSTGRES_PASSWORD=belajar123 -p 5432:5432 -d postgres:16
  ```
- Masuk ke dalam container untuk eksplorasi langsung lewat `bash` dan `psql`

**2. Administrasi Dasar**
- Mengidentifikasi lokasi file konfigurasi inti PostgreSQL:
  - `postgresql.conf` — pengaturan server (listen address, max connections, memory)
  - `pg_hba.conf` — aturan autentikasi client (siapa boleh konek, dari mana, dengan metode apa)
- Membaca dan menginterpretasi parameter aktif seperti `listen_addresses`, `max_connections`, `shared_buffers`

**3. Konfigurasi Client Authentication**
- Menambahkan rule baru di `pg_hba.conf` untuk mengizinkan koneksi eksternal menggunakan metode autentikasi `scram-sha-256`
- Reload konfigurasi tanpa restart penuh (`pg_reload_conf()`), sesuai praktik minim-downtime di production
- Edit file konfigurasi langsung dari terminal menggunakan `nano`

**4. Koneksi Eksternal via GUI Client**
- Setup koneksi DBeaver ke database PostgreSQL yang berjalan di container

### Masalah yang Ditemukan & Cara Menyelesaikannya

#### Masalah 1 — Docker Desktop gagal start: "Virtualization support not detected"
**Root cause:** Windows Features yang dibutuhkan (`Virtual Machine Platform`, `Windows Subsystem for Linux`) belum diaktifkan di level OS, meski virtualisasi sudah enable di BIOS.

**Solusi:**
```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
wsl --update
wsl --set-default-version 2
```
Diikuti restart penuh sistem.

**Pembelajaran:** Virtualisasi butuh dukungan di dua level sekaligus — BIOS/UEFI *dan* OS.

---

#### Masalah 2 — Autentikasi password gagal terus meski password sudah benar
**Gejala:** Setelah mengubah `pg_hba.conf` dari `trust` ke `scram-sha-256`, koneksi dari DBeaver selalu gagal dengan `FATAL: password authentication failed`, padahal koneksi via `psql` langsung dari dalam container berhasil dengan password yang sama.

**Proses investigasi:**
1. Reset ulang password user — tetap gagal
2. Uji koneksi dari `psql` lewat TCP (`psql -h 127.0.0.1`) untuk mensimulasikan jalur yang sama dengan DBeaver — **berhasil**, membuktikan server dan password sudah benar
3. Kesimpulan: masalah ada di sisi client, bukan server

**Pembelajaran:** Kalau `psql` langsung berhasil tapi GUI client gagal dengan kredensial identik, itu sinyal kuat untuk curiga ke jalur koneksi client, bukan konfigurasi server.

---

#### Masalah 3 — Port conflict: dua server PostgreSQL rebutan port 5432
**Gejala:** DBeaver tetap gagal autentikasi bahkan setelah server dan password dipastikan benar via `psql`.

**Proses investigasi:**
```powershell
netstat -ano | findstr :5432
```
Ditemukan **dua PID berbeda** listening di port yang sama:
```powershell
tasklist /FI "PID eq <PID>"
```
Satu proses adalah `com.docker.backend.exe` (Docker, sesuai harapan), tapi proses lain adalah **`postgres.exe` yang berjalan sebagai Windows Service** — instalasi PostgreSQL native (versi 13) yang sebelumnya tidak disadari ada di sistem.

**Root cause:** DBeaver konsisten "dicegat" oleh PostgreSQL native Windows sebelum request sampai ke container Docker.

**Solusi:**
```powershell
sc query type=service | findstr /i postgres    # identifikasi nama service
net stop postgresql-x64-13                      # dari PowerShell as Administrator
```

**Pembelajaran:** Port conflict adalah kelas masalah klasik di administrasi server — dua service berbeda bisa terlihat identik dari luar, tapi punya identitas dan state sepenuhnya terpisah. `netstat` + `tasklist` adalah kombinasi diagnostik pertama yang tepat saat koneksi gagal secara tidak konsisten.

---

## TAHAP 2 — User & Access Management (Role-Based Access Control)

### Konsep

Prinsip inti administrasi akses database: **least privilege** — user hanya diberi akses seminimal yang ia butuhkan untuk menjalankan tugasnya, tidak lebih. Di dunia production, aplikasi hampir tidak pernah diberi akses superuser.

### Yang Dikerjakan

**1. Membuat user individual dengan privilege terbatas**
```sql
CREATE DATABASE toko_online;
CREATE ROLE app_user WITH LOGIN PASSWORD 'app_pass123';
GRANT CONNECT ON DATABASE toko_online TO app_user;
GRANT USAGE ON SCHEMA public TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON produk TO app_user;
```

**2. Verifikasi privilege lewat pengujian langsung**

Login sebagai `app_user` (bukan superuser), lalu diuji terhadap dua kelompok operasi:

| Operasi | Hasil | Keterangan |
|---|---|---|
| `SELECT` | ✅ Berhasil | Sesuai grant |
| `INSERT` | ✅ Berhasil (setelah fix, lihat di bawah) | Sesuai grant |
| `UPDATE` | ✅ Berhasil | Sesuai grant |
| `DROP TABLE` | ❌ Ditolak — `must be owner of table produk` | Bukan owner, tidak boleh drop |
| `CREATE TABLE` | ❌ Ditolak — `permission denied for schema public` | Tidak punya CREATE di schema |

Hasil ini membuktikan langsung, bukan cuma secara teori, bahwa user aplikasi bisa dibatasi untuk baca-tulis data tanpa bisa merusak struktur database.

**3. Role-based access control (grup privilege)**

Alih-alih grant privilege satu-satu ke tiap user aplikasi, dibuat satu role grup yang menampung seluruh privilege, lalu user individual tinggal digabungkan ke grup tersebut:

```sql
CREATE ROLE app_readwrite;  -- tanpa LOGIN, murni sebagai grup
GRANT USAGE ON SCHEMA public TO app_readwrite;
GRANT SELECT, INSERT, UPDATE, DELETE ON produk TO app_readwrite;
GRANT USAGE, SELECT ON SEQUENCE produk_id_seq TO app_readwrite;

CREATE ROLE app_user_2 WITH LOGIN PASSWORD 'app_pass456';
GRANT app_readwrite TO app_user_2;

-- migrasi user lama dari privilege langsung ke grup
GRANT app_readwrite TO app_user;
REVOKE SELECT, INSERT, UPDATE, DELETE ON produk FROM app_user;
REVOKE USAGE, SELECT ON SEQUENCE produk_id_seq FROM app_user;
```

**Hasil arsitektur akhir:**
```
app_readwrite (role grup, tidak bisa login)
   ├── SELECT, INSERT, UPDATE, DELETE ON produk
   ├── USAGE ON SCHEMA public
   └── USAGE, SELECT ON SEQUENCE produk_id_seq
        │
        ├── app_user     (anggota, bisa login)
        └── app_user_2   (anggota, bisa login)
```

Terverifikasi: `app_user_2` (user baru, tidak pernah di-grant langsung) langsung bisa `SELECT`/`INSERT` murni dari keanggotaan grup. `app_user` (privilege langsungnya sudah dicabut) tetap bisa akses normal karena aksesnya sekarang datang dari grup.

**Keuntungan praktis:** menambah user aplikasi baru cukup 2 baris (`CREATE ROLE` + `GRANT app_readwrite`), dan mengubah kebijakan akses untuk seluruh user aplikasi cukup dilakukan sekali di level grup.

### Masalah yang Ditemukan & Cara Menyelesaikannya

#### Masalah 4 — INSERT gagal: "permission denied for sequence produk_id_seq"
**Gejala:** Meski sudah `GRANT INSERT ON produk`, percobaan `INSERT` tetap gagal dengan `permission denied for sequence produk_id_seq`.

**Root cause:** Kolom `SERIAL PRIMARY KEY` di PostgreSQL secara implisit membuat objek **sequence** terpisah untuk auto-increment. Sequence ini adalah objek dengan privilege sendiri, terpisah dari tabelnya — `GRANT` ke tabel tidak otomatis mencakup sequence yang menyertainya.

**Solusi:**
```sql
GRANT USAGE, SELECT ON SEQUENCE produk_id_seq TO app_user;
```
Untuk mencegah masalah ini berulang di setiap tabel baru:
```sql
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_user;
```

**Pembelajaran:** Ini salah satu "gotcha" paling umum bagi yang baru belajar privilege management di PostgreSQL — tabel dengan kolom `SERIAL`/`IDENTITY` selalu punya sequence pendamping yang privilege-nya harus di-grant terpisah.

---

## TAHAP 3 — Backup & Disaster Recovery

### Konsep

Backup yang tidak pernah diuji proses restore-nya sama saja dengan tidak punya backup — asumsi "pasti bisa dipulihkan" itu berisiko, dan baru terbukti benar-salah pada saat paling tidak menguntungkan. Tahap ini fokus pada dua hal: membuat backup dengan format yang tepat sesuai kebutuhan, dan **membuktikan lewat simulasi nyata** bahwa proses pemulihannya benar-benar bekerja.

### Yang Dikerjakan

**1. Backup format plain SQL (`pg_dump`)**
```bash
pg_dump -U postgres -d toko_online -F p -f /tmp/toko_online_backup.sql
```
Format `plain` menghasilkan file `.sql` yang bisa dibaca langsung sebagai teks — berguna untuk diaudit atau diinspeksi manual. Isinya mencakup `CREATE TABLE`, `CREATE SEQUENCE`, data (`COPY`), hingga `GRANT` privilege yang pernah diatur di Tahap 2.

**2. Backup seluruh server (`pg_dumpall`)**
```bash
pg_dumpall -U postgres -f /tmp/full_backup.sql
```
Berbeda dari `pg_dump` yang hanya mencakup satu database, `pg_dumpall` menangkap **semua database sekaligus semua role/user** di server — termasuk `app_user`, `app_readwrite`, `app_user_2` beserta password-nya (tersimpan sebagai hash `SCRAM-SHA-256`, bukan plain text). Penting untuk pemulihan penuh server, karena restore `pg_dump` biasa tidak menyertakan role — bisa menyebabkan error `role does not exist` di server baru.

**3. Backup format custom (`pg_dump -F c`)**
```bash
pg_dump -U postgres -d toko_online -F c -f /tmp/toko_online_backup.dump
```
Format ini terkompresi otomatis dan mendukung fitur lanjutan: restore selektif (per tabel), dan parallel restore untuk database besar. Untuk database sangat kecil (seperti pada latihan ini), overhead struktur metadata format custom bisa membuat ukurannya justru **lebih besar** dari format plain — keuntungan kompresi baru terasa signifikan pada dataset besar.

**4. Simulasi disaster recovery penuh**

Skenario: tabel `produk` terhapus tidak sengaja.
```sql
DROP TABLE produk CASCADE;
```
Diverifikasi dengan `\dt` bahwa tabel benar-benar hilang ("Did not find any relations"), lalu dipulihkan:
```bash
psql -U postgres -d toko_online -f /tmp/toko_online_backup.sql
```
Hasil verifikasi: seluruh data kembali utuh, termasuk riwayat `UPDATE` sebelumnya (harga `Kopi Susu` tetap 20000, bukan kembali ke harga awal 18000) — membuktikan backup menangkap **state data terakhir**, bukan snapshot awal.

**5. Restore dari format custom (`pg_restore`)**
```bash
pg_restore -U postgres -d toko_online --clean --if-exists /tmp/toko_online_backup.dump
```
Diuji dengan skenario drop table yang sama — berhasil memulihkan data 100%.

**6. Restore selektif per tabel**
```bash
pg_restore -U postgres -d toko_online -t produk --clean --if-exists /tmp/toko_online_backup.dump
```
Fitur yang tidak tersedia di format plain — berguna ketika hanya satu tabel yang perlu dipulihkan, tanpa menyentuh objek database lain.

**7. Automated backup dengan cron**

Membuat jadwal backup otomatis harian:
```bash
echo '0 2 * * * pg_dump -U postgres -d toko_online -F c -f /tmp/backup_$(date +\%Y\%m\%d).dump' | crontab -
service cron start
```
Dilengkapi kebijakan retensi (hapus backup lebih tua dari 7 hari) sebagai konsep tambahan untuk mencegah penumpukan file backup tanpa batas.

### Masalah yang Ditemukan & Cara Menyelesaikannya

#### Masalah 5 — `pg_restore` gagal: "relation already exists" dan "duplicate key value"
**Gejala:** Percobaan restore selektif (`pg_restore -t produk`) tanpa opsi tambahan menghasilkan dua error sekaligus:
```
ERROR: relation "produk" already exists
ERROR: duplicate key value violates unique constraint "produk_pkey"
```

**Root cause:** Restore dijalankan ke database yang **masih berisi data** (tabel `produk` sudah ada dari restore sebelumnya). `pg_restore` tanpa `--clean` mencoba membuat ulang objek yang sudah ada, menyebabkan konflik pada `CREATE TABLE` dan `COPY` data.

**Solusi:**
```bash
pg_restore -U postgres -d toko_online -t produk --clean --if-exists /tmp/toko_online_backup.dump
```
`--clean` menghapus objek lama sebelum membuat ulang; `--if-exists` mencegah error tambahan jika objek ternyata belum ada saat proses `--clean` dijalankan.

**Pembelajaran:** Restore ke database kosong vs restore ke database yang sudah terisi sebagian adalah dua skenario berbeda yang butuh penanganan berbeda. Di production, lupa menyertakan `--clean` saat restore ke lingkungan yang datanya sudah ada adalah sumber error yang umum terjadi.

---

## TAHAP 4 — Performance Tuning & Monitoring

### Konsep

Database yang "jalan" dan database yang "cepat" adalah dua hal berbeda. Tahap ini fokus pada kemampuan **membaca cara PostgreSQL mengeksekusi query** (query plan), memahami dampak nyata index terhadap performa, dan menggunakan tool monitoring untuk menemukan query paling lambat tanpa harus menebak-nebak secara manual.

### Yang Dikerjakan

**1. Menyiapkan dataset representatif**

Tabel `produk` yang sebelumnya hanya berisi 2 baris diisi 100.000 baris data dummy menggunakan `generate_series`, agar efek performa index terlihat nyata (dataset kecil membuat semua query terasa cepat, sehingga sulit membandingkan).

**2. Membaca query plan dengan `EXPLAIN ANALYZE`**

Perbandingan langsung query `WHERE nama = ...` sebelum dan sesudah index dibuat, pada dataset yang sama (100.002 baris):

| | Tanpa Index (Seq Scan) | Dengan Index (Index Scan) |
|---|---|---|
| Rows diperiksa | 100.001 dari 100.002 baris | Langsung ke baris yang tepat |
| Execution Time | 6.487 ms | **0.078 ms** |
| Speedup | — | **~83x lebih cepat** |

```sql
CREATE INDEX idx_produk_nama ON produk(nama);
```

**3. Memahami strategi planner yang berbeda-beda**

Setelah index kedua dibuat di kolom `harga`, dua query berbeda menunjukkan **dua strategi eksekusi yang berbeda** meski sama-sama memakai index yang sama:

- `WHERE harga > 900` (hasil ~9.800 baris, ~10% dari tabel) → **Bitmap Heap Scan**, strategi hybrid yang mengumpulkan lokasi baris lewat index lalu membaca heap sekaligus — lebih efisien untuk hasil besar dibanding Index Scan murni.
- `ORDER BY harga DESC LIMIT 10` (hasil kecil, cuma 10 baris) → **Index Scan Backward**, mengambil langsung dari nilai terbesar tanpa perlu mengurutkan seluruh tabel — 0.191 ms.

**Insight kunci:** PostgreSQL memakai *cost-based optimizer* — keberadaan index tidak otomatis berarti "Index Scan" akan selalu dipakai. Planner memilih strategi berdasarkan estimasi jumlah baris yang cocok; untuk hasil yang mencakup porsi besar tabel, Seq Scan atau Bitmap Scan bisa jadi lebih efisien daripada Index Scan murni.

**4. Setup `pg_stat_statements` untuk monitoring query**

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```
Setelah didaftarkan di `shared_preload_libraries` dan container di-restart, ekstensi ini mencatat statistik agregat setiap jenis query yang dijalankan (dengan literal dinormalisasi menjadi placeholder `$1`), termasuk jumlah pemanggilan (`calls`) dan total waktu eksekusi (`total_exec_time`).

Hasil sampel setelah menjalankan beberapa query:

| Query | Calls | Total Time (ms) | Catatan |
|---|---|---|---|
| `SELECT COUNT(*) FROM produk` | 1 | 65.6 | Paling lambat — scan seluruh tabel untuk menghitung |
| `SELECT * ... ORDER BY harga DESC LIMIT $1` | 1 | 12.6 | Lambat karena `harga` belum ada index saat itu |
| `SELECT * WHERE harga > $1` | 1 | 12.3 | Sama, belum ada index |
| `SELECT * WHERE nama = $1` | 4 | 0.7 | Dipanggil 4x tapi total waktu lebih kecil — efek index `idx_produk_nama` |

Berdasarkan data ini, index kedua (`idx_produk_harga`) dibuat sebagai tindak lanjut langsung dari temuan monitoring — bukan tebakan, melainkan keputusan berbasis data.

### Masalah yang Ditemukan & Cara Menyelesaikannya

#### Masalah 6 — Sequence hilang pasca restore selektif, menyebabkan INSERT gagal
**Gejala:** Saat mencoba mengisi data dummy dalam jumlah besar, `INSERT` gagal dengan `null value in column "id" violates not-null constraint`.

**Proses investigasi:**
1. Cek struktur tabel (`\d produk`) — kolom `id` ternyata tidak lagi memiliki `DEFAULT nextval(...)`
2. Percobaan memperbaiki `DEFAULT` langsung gagal dengan error `relation "produk_id_seq" does not exist`
3. Cek daftar sequence (`\ds`) — benar-benar kosong, sequence sudah tidak ada sama sekali

**Root cause:** Ini efek samping dari latihan Tahap 3 — restore selektif per tabel (`pg_restore -t produk`) hanya memulihkan objek tabel itu sendiri, **tidak menyertakan sequence pendamping** yang terhubung ke kolom `SERIAL`. Sequence adalah objek independen di katalog PostgreSQL, sehingga tidak otomatis ikut ter-restore hanya karena terhubung secara logis ke satu kolom tabel.

**Solusi:**
```sql
CREATE SEQUENCE produk_id_seq;
ALTER TABLE produk ALTER COLUMN id SET DEFAULT nextval('produk_id_seq');
ALTER SEQUENCE produk_id_seq OWNED BY produk.id;
SELECT setval('produk_id_seq', (SELECT MAX(id) FROM produk));
```
`ALTER SEQUENCE ... OWNED BY` penting agar sequence ikut terhapus otomatis jika tabelnya suatu saat di-drop, dan `setval` memastikan penomoran berikutnya tidak bentrok dengan data yang sudah ada.

**Pembelajaran:** Ini melengkapi pembelajaran privilege sequence dari Tahap 2 — sequence bukan cuma punya privilege terpisah dari tabelnya, tapi juga **objek fisik terpisah** yang harus ditangani secara eksplisit di setiap operasi backup, restore, maupun grant. Restore selektif per tabel harus dipastikan mencakup seluruh objek pendukungnya, bukan cuma tabel utamanya.

---

## TAHAP 5 — High Availability & Replikasi

### Konsep

Server database tunggal adalah *single point of failure* — kalau server itu mati, seluruh aplikasi ikut down. **Streaming replication** menyelesaikan ini dengan menjalankan satu server **primary** (menerima semua tulisan) dan satu atau lebih **replica** (menyalin perubahan data secara real-time), sehingga replica siap "naik jabatan" menjadi primary baru saat dibutuhkan (failover).

### Yang Dikerjakan

**1. Konfigurasi primary untuk mendukung replikasi**

Parameter ditambahkan ke `postgresql.conf` (membutuhkan restart karena termasuk kategori `change requires restart`):
```
wal_level = replica
max_wal_senders = 5
```
Serta rule khusus replikasi di `pg_hba.conf`:
```
host    replication     all             0.0.0.0/0               md5
```

**2. User khusus replikasi**
```sql
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD '...';
```
Diverifikasi lewat `\du replicator` — muncul dengan attribute **Replication**, terpisah dari role aplikasi biasa yang dibuat di Tahap 2.

**3. Jaringan antar container**

Dua container terpisah (`pg-dba` sebagai primary, `pg-replica` sebagai replica baru) perlu saling berkomunikasi memakai nama container, bukan `localhost`. Ini memerlukan **user-defined network**, bukan default bridge network Docker:
```bash
docker network create pg-network
docker network connect pg-network pg-dba
docker network connect pg-network pg-replica
```

**4. Sinkronisasi awal dengan `pg_basebackup`**
```bash
pg_basebackup -h pg-dba -U replicator -D /var/lib/postgresql/data -Fp -Xs -P -R
```
Opsi `-R` secara otomatis membuat `standby.signal` dan konfigurasi `primary_conninfo` — dua artefak yang membuat PostgreSQL mengenali dirinya sebagai standby saat pertama kali dinyalakan. Basebackup berhasil menyalin seluruh ~41 MB data (termasuk 100.000+ baris tabel `produk` dan index dari Tahap 4) dengan sempurna.

**5. Verifikasi replikasi real-time**

| Pengujian | Hasil |
|---|---|
| Primary mendeteksi replica (`pg_stat_replication`) | 1 baris, `state = streaming`, `sync_state = async` |
| `INSERT` baru di primary | Berhasil |
| Baris baru otomatis muncul di replica (tanpa insert manual) | **Berhasil**, muncul instan dengan `id` yang konsisten |
| `INSERT` langsung ke replica | **Ditolak** — `cannot execute INSERT in a read-only transaction` |

**6. Simulasi failover manual**

Skenario: primary mati total.
```bash
docker stop pg-dba
```
Status replica sebelum promosi dicek terlebih dahulu:
```sql
SELECT pg_is_in_recovery();  -- hasil: t (true, masih standby)
```
Replica dipromosikan menjadi primary baru:
```sql
SELECT pg_promote();  -- hasil: t (true)
```
Status dicek ulang setelah promosi:
```sql
SELECT pg_is_in_recovery();  -- hasil: f (false, sudah jadi primary)
```
Diverifikasi dengan `INSERT` langsung ke bekas-replica — berhasil (`INSERT 0 1`), membuktikan instance tersebut kini benar-benar berfungsi sebagai primary, bukan lagi read-only. Kemunculan proses `autovacuum launcher` dan `logical replication launcher` di daftar proses (proses yang hanya berjalan di primary) menjadi konfirmasi tambahan di level sistem.

### Masalah yang Ditemukan & Cara Menyelesaikannya

#### Masalah 7 — Container tidak bisa start setelah data directory dikosongkan manual
**Gejala:** Setelah mencoba membersihkan data directory replica dengan `rm -rf /var/lib/postgresql/data/*` (untuk mengulang proses dari awal), container gagal start dengan error `could not open file "postmaster.pid": No such file or directory`, diikuti `performing immediate shutdown`.

**Root cause:** Menghapus isi data directory secara manual pada container yang sebelumnya sudah pernah diinisialisasi membuat PostgreSQL berada pada kondisi ambigu — direktorinya ada tapi kosong, dan entrypoint container tidak tahu apakah harus melakukan inisialisasi baru atau menganggap ini instalasi yang rusak.

**Solusi:** Menghapus total container (`docker rm`) dan membuat ulang dari image dasar dengan `sleep infinity` sebagai perintah awal (menunda start PostgreSQL), sehingga `pg_basebackup` bisa mengisi data directory dari kondisi benar-benar bersih sebelum PostgreSQL pertama kali dijalankan secara manual.

**Pembelajaran:** Membersihkan data directory PostgreSQL secara manual dengan `rm -rf` bukan cara yang aman untuk "reset" instance — lebih baik memulai dari container/instance baru sepenuhnya.

---

#### Masalah 8 — `pg_basebackup` gagal: "could not translate host name"
**Gejala:** Percobaan pertama `pg_basebackup -h pg-dba ...` gagal dengan `could not translate host name "pg-dba" to address: Name or service is not known`, meskipun kedua container sama-sama berjalan di Docker.

**Root cause:** Container yang dibuat dengan `docker run` tanpa spesifikasi network secara default masuk ke **default bridge network** Docker, yang **tidak menyediakan DNS resolution otomatis berdasarkan nama container** — fitur ini hanya tersedia di *user-defined network*.

**Solusi:**
```bash
docker network create pg-network
docker network connect pg-network pg-dba
docker network connect pg-network pg-replica
```
`docker network connect` bisa dijalankan pada container yang sudah berjalan tanpa perlu restart atau kehilangan data — kedua container menjadi bisa saling resolve nama satu sama lain begitu berada di network yang sama.

**Pembelajaran:** Komunikasi antar-container berdasarkan nama hanya bekerja pada network yang dibuat secara eksplisit. Ini detail infrastruktur yang mudah terlewat ketika sudah terbiasa menjalankan container tunggal (seperti pada Tahap 1–4), tapi menjadi krusial begitu arsitektur melibatkan lebih dari satu instance yang perlu saling berkomunikasi.

---

#### Masalah 9 — PostgreSQL menolak start: "Permission denied" pada file konfigurasi
**Gejala:** Setelah `pg_basebackup` berhasil dan PostgreSQL dicoba dijalankan secara manual di replica, muncul error `could not open configuration file "postgresql.conf": Permission denied`.

**Root cause:** `pg_basebackup` dijalankan melalui `docker exec` sebagai user default (`root`), sehingga seluruh file hasil salinan menjadi milik `root`. PostgreSQL menolak untuk berjalan sebagai user `postgres` ketika tidak memiliki akses baca ke direktori datanya sendiri — ini perilaku keamanan yang disengaja, bukan bug.

**Solusi:**
```bash
chown -R postgres:postgres /var/lib/postgresql/data
chmod 700 /var/lib/postgresql/data
```
Setelah kepemilikan dan permission diperbaiki, PostgreSQL berhasil start dan langsung masuk mode standby, dilanjutkan dengan log `started streaming WAL from primary`.

**Pembelajaran:** Operasi yang melibatkan `docker exec` tanpa spesifikasi user (`-u postgres`) berjalan sebagai `root` secara default. Untuk proses yang berkaitan langsung dengan data directory PostgreSQL, ketidaksesuaian ownership adalah sumber error yang umum, dan PostgreSQL sengaja bersikap ketat soal ini sebagai lapisan keamanan tambahan terhadap data directory yang berisi seluruh isi database.

---

## TAHAP 6 — Simulasi Skenario Production

### Konsep

Tahap penutup ini tidak memperkenalkan konsep baru, melainkan menggabungkan seluruh kompetensi dari Tahap 1–5 lewat tiga skenario "hari buruk" yang umum dialami DBA di dunia nyata: kehabisan slot koneksi, deadlock antar-transaksi, dan kebutuhan akan connection pooling di skala besar.

### Yang Dikerjakan

**1. Simulasi "Too Many Connections"**

`max_connections` diturunkan sementara ke nilai kecil (5) untuk memicu kondisi nyata tanpa perlu benar-benar membuka ratusan koneksi:
```sql
-- postgresql.conf
max_connections = 5
```
Enam sesi `psql` dibuka bersamaan — percobaan ke-6 gagal dengan `FATAL: sorry, too many clients already`, muncul lebih cepat dari yang diperkirakan karena PostgreSQL secara default menyisakan slot untuk `superuser_reserved_connections`.

Diagnosis dilakukan dengan query yang menjadi langkah pertama standar DBA saat menghadapi masalah koneksi:
```sql
SELECT pid, usename, application_name, client_addr, state, query FROM pg_stat_activity;
```
Hasilnya menunjukkan beberapa koneksi berstatus `idle` — pola yang identik dengan *connection leak* di production (aplikasi yang lupa menutup koneksi). Solusi darurat diterapkan dengan memutus koneksi bermasalah secara paksa:
```sql
SELECT pg_terminate_backend(<pid>);
```
Diverifikasi berhasil lewat pengecekan ulang `pg_stat_activity` — koneksi yang ditarget benar-benar hilang dari daftar.

**2. Simulasi Deadlock**

Dua sesi `psql` terpisah masing-masing memulai transaksi dan mengunci baris berbeda:
```sql
-- Sesi A
BEGIN; UPDATE produk SET harga = 25000 WHERE id = 1;
-- Sesi B
BEGIN; UPDATE produk SET harga = 30000 WHERE id = 2;
```
Lalu masing-masing mencoba mengunci baris yang sudah dipegang sesi lain — memicu lingkaran saling tunggu:
```sql
-- Sesi A mencoba id=2 (dipegang B) → menggantung
-- Sesi B mencoba id=1 (dipegang A) → memicu deadlock
```
PostgreSQL secara otomatis mendeteksi pola melingkar ini dan membatalkan salah satu transaksi:
```
ERROR: deadlock detected
DETAIL: Process 62 waits for ShareLock on transaction 813; blocked by process 53.
Process 53 waits for ShareLock on transaction 814; blocked by process 62.
```
Sesi yang "menang" (A) langsung lanjut begitu sesi yang dikorbankan (B) dibatalkan. Data akhir diverifikasi konsisten dan bisa ditelusuri sepenuhnya — kedua perubahan dari sesi A berhasil ter-commit, sementara kedua percobaan dari sesi B gagal total (baik yang dibatalkan otomatis maupun rollback manual).

**3. Setup PgBouncer (Connection Pooling)**

Container PgBouncer dijalankan pada network yang sama dengan primary, dikonfigurasi untuk meneruskan koneksi ke `toko_online`:
```bash
docker run --name pgbouncer --network pg-network \
  -e DB_HOST=pg-dba -e DB_PORT=5432 -e DB_USER=postgres -e DB_PASSWORD=... \
  -e DB_NAME=toko_online -e POOL_MODE=transaction \
  -e MAX_CLIENT_CONN=100 -e DEFAULT_POOL_SIZE=5 \
  -e AUTH_TYPE=plain -p 6432:5432 -d edoburu/pgbouncer
```

Setelah berhasil terhubung, pembuktian inti connection pooling dilakukan: tiga sesi client dibuka bersamaan melalui PgBouncer, lalu dibandingkan dengan koneksi fisik sesungguhnya di PostgreSQL:

| Sisi | Jumlah Koneksi |
|---|---|
| Client yang terhubung lewat PgBouncer | 3 |
| Koneksi fisik nyata ke PostgreSQL (`pg_stat_activity`) | 2 (1 idle, 1 active) |

Hasil ini membuktikan `pool_mode: transaction` bekerja sesuai desain — PgBouncer memakai ulang (reuse) koneksi fisik yang sama untuk melayani beberapa client, alih-alih membuka satu koneksi fisik terpisah per client seperti pada koneksi langsung ke PostgreSQL.

### Masalah yang Ditemukan & Cara Menyelesaikannya

#### Masalah 10 — PgBouncer gagal start: environment variable tidak sesuai
**Gejala:** Percobaan pertama menjalankan container PgBouncer (`edoburu/pgbouncer`) dengan variabel `DATABASES_HOST`, `DATABASES_PORT`, dst gagal dengan error `Setup pgbouncer config error! You must set DB_HOST env`.

**Root cause:** Nama environment variable yang dipakai tidak sesuai dengan konvensi yang diharapkan image tersebut (`DB_HOST`, bukan `DATABASES_HOST`).

**Solusi:** Container dibuat ulang dengan nama variabel yang benar (`DB_HOST`, `DB_PORT`, `DB_USER`, dst).

**Pembelajaran:** Setiap image Docker punya konvensi environment variable sendiri yang harus diverifikasi dari dokumentasi resmi image tersebut, bukan diasumsikan sama antar image sejenis.

---

#### Masalah 11 — Port mapping tidak sesuai dengan port listen aktual PgBouncer
**Gejala:** Container berhasil start, tapi koneksi ke port yang dimapping tidak berhasil terhubung sama sekali.

**Root cause:** Log startup PgBouncer menunjukkan dia mendengarkan di port `5432` secara internal (bukan `6432` seperti yang diasumsikan), sehingga mapping `-p 6432:6432` mengarah ke port yang salah di dalam container.

**Solusi:** Port mapping diperbaiki menjadi `-p 6432:5432` (format `HOST_PORT:CONTAINER_PORT`), sehingga akses eksternal tetap lewat port 6432 tapi diarahkan ke port internal yang benar.

**Pembelajaran:** Verifikasi port yang benar-benar didengarkan sebuah container harus dilakukan lewat log, bukan diasumsikan dari environment variable yang tersedia — tidak semua image mengekspos port listen sebagai variabel yang bisa dikonfigurasi.

---

#### Masalah 12 — Autentikasi SCRAM gagal khusus untuk koneksi ke database asli
**Gejala:** Koneksi ke database administratif PgBouncer (`db=pgbouncer`) berhasil, tapi koneksi ke database data asli (`db=toko_online`) konsisten gagal dengan `FATAL: SASL authentication failed`, meski password yang dimasukkan benar.

**Root cause:** Berdasarkan pola pada log PgBouncer, autentikasi SCRAM-SHA-256 memerlukan PgBouncer meneruskan (proxy) proses handshake SCRAM secara utuh antara client dan PostgreSQL asli — sesuatu yang tidak didukung penuh oleh versi image yang dipakai untuk koneksi non-administratif.

**Solusi:** Untuk keperluan latihan, `AUTH_TYPE` diturunkan ke `plain` sebagai jalan pragmatis membuktikan konsep pooling tanpa terjebak berlarut-larut pada detail kompatibilitas SCRAM.

**Pembelajaran:** Ini catatan penting untuk konteks production sungguhan: `auth_type: plain` mengirim password tanpa hashing di jalur client-ke-pooler, dan **tidak direkomendasikan** di luar lingkungan lab tertutup. Solusi production yang lebih tepat adalah menggunakan `auth_type: md5` dengan `userlist.txt` berisi hash MD5 eksplisit, atau memastikan versi PgBouncer yang dipakai mendukung SCRAM pass-through secara penuh — sebuah pengingat bahwa tidak semua kombinasi versi software kompatibel satu sama lain meski konsepnya sama-sama "mendukung SCRAM".

---

## Skill yang Terlatih

- Instalasi dan konfigurasi Docker Desktop di Windows (WSL2 backend)
- Navigasi dan administrasi PostgreSQL dari command line (`psql`)
- Membaca dan mengedit `postgresql.conf` dan `pg_hba.conf`
- Edit file konfigurasi lewat terminal menggunakan `nano`
- Reload konfigurasi tanpa downtime (`pg_reload_conf`)
- Konsep dan implementasi metode autentikasi PostgreSQL (`trust`, `scram-sha-256`, `md5`)
- Debugging sistematis: mengisolasi masalah server vs client, diagnosis port conflict (`netstat`, `tasklist`)
- Prinsip least privilege dan implementasinya dengan `GRANT`/`REVOKE`
- Perbedaan privilege tabel vs sequence
- Role-based access control — role sebagai grup vs role sebagai user login
- Verifikasi privilege lewat pengujian langsung (positive & negative testing)
- Backup database dengan `pg_dump` (format plain & custom) dan `pg_dumpall`
- Restore penuh maupun selektif per tabel dengan `psql -f` dan `pg_restore`
- Simulasi disaster recovery end-to-end (kehilangan data → pemulihan → verifikasi)
- Automated backup terjadwal dengan cron, termasuk kebijakan retensi
- Membaca dan menginterpretasi query plan (`EXPLAIN ANALYZE`)
- Membedakan strategi eksekusi: Seq Scan, Index Scan, Index Scan Backward, Bitmap Heap Scan
- Membuat index dan mengukur dampaknya secara kuantitatif terhadap performa query
- Monitoring query dengan `pg_stat_statements`, termasuk konfigurasi `shared_preload_libraries`
- Mengambil keputusan optimasi (pembuatan index) berdasarkan data monitoring, bukan tebakan
- Konfigurasi streaming replication PostgreSQL (`wal_level`, `max_wal_senders`, role replikasi)
- Setup Docker user-defined network untuk komunikasi antar-container berdasarkan nama
- Sinkronisasi data awal antar instance dengan `pg_basebackup`
- Verifikasi replikasi real-time dan sifat read-only pada replica
- Simulasi failover manual (`pg_promote`, `pg_is_in_recovery`)
- Debugging permission ownership pada data directory PostgreSQL di lingkungan container
- Diagnosis dan pemulihan situasi "too many connections" (`pg_stat_activity`, `pg_terminate_backend`)
- Memahami dan mensimulasikan deadlock, termasuk cara PostgreSQL mendeteksi dan menyelesaikannya secara otomatis
- Setup dan konfigurasi connection pooling dengan PgBouncer (pool mode, port mapping, autentikasi)
- Membaca log container untuk diagnosis startup dan autentikasi

---

## Catatan Reflektif

Enam tahap ini menunjukkan bahwa pekerjaan DBA bukan cuma soal menghafal syntax SQL, tapi soal **berpikir dalam kerangka keamanan, skalabilitas, kesiapan menghadapi kegagalan, efisiensi, dan kelangsungan layanan** — dan pada akhirnya, kemampuan menghadapi situasi tak terduga dengan proses diagnosis yang sistematis, bukan tebakan. Siapa yang seharusnya bisa melakukan apa, bagaimana membuktikan batasan itu benar-benar bekerja, bagaimana memastikan data bisa dipulihkan saat skenario terburuk terjadi, bagaimana mengetahui bagian mana dari sistem yang menjadi hambatan performa, bagaimana memastikan sistem tetap melayani pengguna ketika komponen utamanya gagal, dan bagaimana menangani lonjakan koneksi maupun konflik transaksi tanpa mengorbankan integritas data — semuanya dipraktikkan secara langsung, bukan sekadar dibaca dari teori.

Dua belas masalah yang ditemukan dan diselesaikan sepanjang enam tahap ini — mulai dari port conflict, kesalahpahaman jalur autentikasi client vs server, objek sequence yang privilege dan eksistensinya terpisah dari tabel, error restore akibat objek yang sudah ada, DNS resolution antar-container, permission ownership pada proses replikasi, hingga ketidakcocokan metode autentikasi pada connection pooler — semuanya adalah jenis masalah yang secara realistis muncul di lingkungan production, bukan skenario buatan untuk latihan. Beberapa di antaranya bahkan saling terhubung lintas tahap (masalah sequence di Tahap 4 adalah konsekuensi keputusan restore di Tahap 3), sebuah pengingat bahwa operasi database jarang berdiri sendiri — keputusan teknis di satu area kerap memunculkan efek samping di area lain yang baru terlihat belakangan.

Latihan ini dibangun murni lewat proses coba-gagal-diagnosis-perbaiki yang konsisten di setiap tahap, bukan mengikuti jalur yang selalu mulus — dan justru itulah yang membuatnya representatif dengan pekerjaan DBA yang sesungguhnya.

---

*Roadmap latihan PostgreSQL DBA — 6 dari 6 tahap selesai: Instalasi & Administrasi Dasar, User & Access Management, Backup & Restore, Performance & Monitoring, High Availability & Replikasi, dan Simulasi Skenario Production.*
