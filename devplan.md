# Development Plan — Modul Penjadwalan Tes Seleksi & Wawancara PMB
## Vibe Coding & Venture SEVIMA
**Nama:** Arda Bahrul Ulum
**Repo:** `admission-app` (fork) — pengembangan lanjutan di atas sistem PMB yang sudah berjalan
**Tema:** Menambahkan modul penjadwalan tes ke aplikasi PMB existing (bukan dari nol)

---

## Konteks Sistem yang Sudah Ada (ringkas)

Aplikasi PMB sudah berjalan penuh dengan stack:
- **Frontend:** React 18 + Vite + Tailwind CSS (routing via window, tanpa react-router)
- **Backend:** Laravel 12 + Sanctum (auth admin), SQLite (dev) / PostgreSQL (prod)
- **Komunikasi:** Fetch API + custom wrapper (`utils/api.js`), token di `sessionStorage`

Fitur yang **sudah selesai** (tidak akan dibangun ulang):
pendaftaran online, generate nomor (`PMB-2025-XXXX`), cek status, dashboard admin + statistik per prodi/jalur, update status, login Sanctum, export CSV, heregistrasi.

Tabel existing: `pendaftars` (kolom: `id, nomor_pendaftaran, nama, nomor_hp, email, asal_sekolah, prodi, jalur, status, heregistrasi_at`) dan `users` (admin). Status pendaftar: `Menunggu` / `Lolos Seleksi` / `Tidak Lolos`.

> **Titik integrasi kunci:** modul penjadwalan **membaca** pendaftar berstatus `Lolos Seleksi` dari tabel `pendaftars` yang sudah ada.

---

# BAGIAN 1 — Analisa Teknis

## 1.1 Identifikasi Pengguna

| Pengguna | Peran dalam modul penjadwalan |
|----------|-------------------------------|
| **Admin PMB** *(existing)* | Peran **baru**: membuat sesi tes (tanggal, jam, ruang, kuota), menugaskan peserta yang lolos ke sesi, mereview & menyetujui/menolak permintaan reschedule, melihat rekap kehadiran. |
| **Calon Mahasiswa / Peserta** *(existing)* | Peran **baru**: melihat jadwal tes pribadinya, men-download kartu tes, dan mengajukan permintaan reschedule jika berhalangan. |
| **Operator / Panitia Lapangan** *(pengguna baru — khusus modul ini)* | Mencatat kehadiran peserta di hari-H berdasarkan nomor pendaftaran/QR, melihat daftar peserta per sesi yang ia tangani. Tidak punya akses ke modul pendaftaran/seleksi. |
| **Sistem (aktor otomatis)** | Auto-assign slot saat kuota tersedia, mengubah status sesi `penuh` saat kuota habis, mengirim notifikasi/reminder, dan mencatat audit perubahan jadwal. |

Catatan: Admin & Calon Mahasiswa sudah ada di sistem pendaftaran, tetapi **Operator Lapangan adalah role baru** yang spesifik untuk modul penjadwalan — ia tidak perlu (dan tidak boleh) melihat seluruh data seleksi.

## 1.2 Fitur Utama per Pengguna

**Admin PMB**
1. CRUD **sesi tes** (jenis: Tes Tulis / Wawancara; tanggal, jam mulai–selesai, ruang/lokasi, kuota).
2. Assign peserta berstatus `Lolos Seleksi` ke sesi (manual pilih, atau auto-distribute berdasarkan kuota & prodi).
3. Dashboard rekap: jumlah peserta terjadwal vs belum terjadwal, kuota terpakai per sesi, rekap kehadiran.
4. Approve / reject permintaan reschedule peserta.
5. Broadcast / resend notifikasi jadwal ke peserta sebuah sesi.

**Calon Mahasiswa / Peserta**
1. Halaman "Jadwal Saya" — input nomor pendaftaran → tampilkan jenis tes, tanggal, jam, ruang.
2. Download / tampilkan **kartu tes** (berisi nomor pendaftaran, nama, jadwal, lokasi).
3. Ajukan **permintaan reschedule** dengan memilih slot alternatif + alasan.
4. Lihat status permintaan reschedule (Diajukan / Disetujui / Ditolak).

**Operator Lapangan**
1. Buka daftar peserta per sesi yang ditugaskan.
2. Tandai **kehadiran** (Hadir / Tidak Hadir) per peserta saat check-in.
3. Pencarian cepat peserta dalam sesi by nomor pendaftaran/nama.

> Semua fitur di atas **belum ada** di sistem saat ini. Fokus pada masalah dari brief: jadwal tidak tersampaikan, peserta tidak hadir, informasi tidak terpusat.

## 1.3 Tech Stack Tambahan

Stack utama **tidak diganti** (React 18 + Tailwind, Laravel 12). Komponen tambahan:

| Kebutuhan | Pilihan | Alasan |
|-----------|---------|--------|
| Date/time picker (form sesi) | `react-day-picker` + util `Intl.DateTimeFormat` | Ringan, headless, mudah di-styling dengan Tailwind, tanpa CSS framework tambahan; konsisten dengan aturan "Tailwind only". |
| Penjadwalan kirim reminder | **Laravel Queue** (driver `database`) + **Scheduler** (`schedule:run` via cron) | Laravel sudah ada tabel `jobs` (migration `create_jobs_table` sudah ada). Reminder H-1 cukup dijalankan via scheduled job, tanpa service eksternal. |
| Channel notifikasi | **Laravel Notification** (mail driver `log`/SMTP) | Notification class native Laravel; di dev pakai driver `log`, di prod tinggal ganti SMTP. Tidak perlu integrasi pihak ketiga untuk prototype. |
| Kartu tes / QR check-in | `qrcode.react` (frontend) | Generate QR berisi nomor pendaftaran untuk mempercepat check-in operator; client-side, tanpa dependency backend tambahan. |
| Tabel data admin | Reuse pola `TabelPendaftar.jsx` existing | Tidak menambah library tabel; konsistensi UI & menghindari over-engineering. |

> Prinsip: minimalkan dependency baru, manfaatkan kapabilitas Laravel (Queue/Scheduler/Notification) yang sudah tersedia di framework.

## 1.4 Batasan & Asumsi

1. **Hanya peserta `Lolos Seleksi` yang dapat dijadwalkan.** Modul tidak mengubah logika status seleksi; ia hanya **membaca** `pendaftars.status = 'Lolos Seleksi'` sebagai sumber peserta. Perubahan di modul pendaftaran tidak boleh tersentuh.
2. **Tidak mengubah skema tabel `pendaftars` atau `users`.** Relasi dibuat lewat foreign key di tabel baru (`pendaftar_id`, `user_id`), sehingga fitur lama (pendaftaran, cek status, dashboard) tetap berjalan tanpa regresi.
3. **Satu peserta = satu jadwal aktif per jenis tes** (1 Tes Tulis + 1 Wawancara). Reschedule mengganti slot, bukan menambah jadwal ganda.
4. **Otentikasi admin tetap pakai Sanctum existing.** Role Operator diasumsikan diberi token Sanctum juga, dibedakan lewat kolom `role` di `users` (ditambahkan via migration baru yang nullable, default `admin`, agar data user lama tetap valid).
5. **Notifikasi di dev menggunakan mail driver `log`** (tidak benar-benar mengirim email/WA). Asumsi: integrasi WA/SMTP gateway nyata di luar scope prototype.
6. **Timezone tunggal** `Asia/Jakarta`; tidak ada penanganan multi-timezone.

---

# BAGIAN 2 — Bisnis Proses & Flow

## 2.1 Flow Utama: Penjadwalan Tes Seleksi

```
[Admin] → Login (Sanctum, existing) → [Terotentikasi]
[Admin] → Buat Sesi Tes (jenis, tanggal, jam, ruang, kuota) → [sesi_tes tersimpan, status=aktif]
       ↓
[Admin] → Buka daftar peserta "Lolos Seleksi" (BACA dari tabel pendaftars existing)
       ↓ jika [peserta belum punya jadwal untuk jenis tes ini]
[Admin] → Assign peserta ke sesi (manual / auto-distribute)
       ↓
[Sistem] → Validasi kuota sesi
       ↓ jika [kuota masih tersedia]
[Sistem] → Buat record jadwal_peserta (pendaftar_id → sesi_tes_id) → [jadwal terbuat]
       ↓ jika [kuota sesi habis setelah assign]
[Sistem] → Ubah sesi_tes.status = 'penuh'
       ↓
[Sistem] → Trigger Notification (queue) → [peserta diberi tahu jadwalnya]
       ↓
[Peserta] → Buka "Jadwal Saya" (input nomor pendaftaran) → [jadwal + kartu tes tampil]
       ↓
[Operator] → Hari-H: check-in peserta by nomor pendaftaran/QR → [absensi.status = Hadir]
```

**Titik integrasi:** langkah "Buka daftar peserta Lolos Seleksi" = operasi **READ** ke `pendaftars`. Langkah "Buat record jadwal_peserta" = **WRITE** ke tabel baru yang ber-FK ke `pendaftars.id`. Sistem lama tidak ditulis ulang.

## 2.2 Flow Alternatif: Peserta Minta Reschedule

```
[Peserta] → Buka "Jadwal Saya" → klik "Ajukan Reschedule"
[Peserta] → Pilih sesi alternatif (yang masih ada kuota) + isi alasan → [permintaan_reschedule tersimpan, status=Diajukan]
       ↓
[Sistem] → Notifikasi ke Admin ada permintaan baru
       ↓
[Admin] → Review permintaan
       ↓ jika [disetujui & slot baru masih ada kuota]
[Admin] → Approve → [Sistem pindahkan jadwal_peserta ke sesi baru, kuota lama -1 / kuota baru +1, status=Disetujui]
       ↓ jika [ditolak / slot penuh]
[Admin] → Reject (beri alasan) → [jadwal_peserta tetap, status permintaan=Ditolak]
       ↓
[Sistem] → Notifikasi hasil ke peserta
```

## 2.3 Happy Path vs Error Path (untuk Flow 2.1)

**Happy path:** Admin buat sesi → assign peserta lolos → kuota cukup → jadwal terbuat → notifikasi terkirim → peserta lihat jadwal → operator catat Hadir. Semua langkah sukses tanpa konflik.

**Error path:**
1. **Kuota habis saat assign** — Sistem menolak assignment, mengembalikan response `{"success": false, "message": "Kuota sesi sudah penuh"}` (HTTP 422), dan menyarankan admin memilih sesi lain. Tidak ada record dibuat.
2. **Peserta sudah punya jadwal untuk jenis tes yang sama** — Sistem menolak (constraint `UNIQUE(pendaftar_id, jenis_tes)`), response 409 `"Peserta sudah memiliki jadwal untuk tes ini"`, mencegah jadwal ganda.
3. *(tambahan)* **Peserta bukan berstatus `Lolos Seleksi`** — divalidasi sebelum insert; response 422 `"Hanya peserta yang lolos seleksi dapat dijadwalkan"`.

---

# BAGIAN 3 — Alur Data

## 3.1 Alur Data: Proses Penjadwalan (Admin)

```
[Admin input form sesi (React: FormSesiTes.jsx)]
  → [POST /api/jadwal/sesi  → Laravel: JadwalSesiController@store]
  → [Validasi StoreSesiRequest → simpan ke tabel sesi_tes]
  → [Admin assign: POST /api/jadwal/assign → JadwalPesertaController@store]
  → [READ tabel pendaftars (cek status=Lolos Seleksi) + cek kuota sesi_tes]
  → [WRITE tabel jadwal_peserta (FK pendaftar_id, sesi_tes_id)]
  → [Dispatch JadwalNotification ke queue (tabel jobs)]
  → [Response JSON {success:true,...} → React update tabel di Admin dashboard]
```

## 3.2 Alur Data: Peserta Cek Jadwal

```
[Peserta input nomor pendaftaran (React: JadwalSaya.jsx)]
  → [GET /api/jadwal/peserta/{nomorPendaftaran} → JadwalPesertaController@show]
  → [Laravel JOIN: pendaftars ⨝ jadwal_peserta ⨝ sesi_tes]
  → [Ambil jenis tes, tanggal, jam, ruang]
  → [Response JSON → React render kartu tes + tombol Reschedule]
```

Layer teknis yang terlibat (sesuai stack existing): **React component → utils/api.js (Fetch) → route api.php → Laravel Controller → Eloquent Model → tabel SQLite/PostgreSQL**, lalu kembali sebagai JSON. Endpoint cek jadwal bersifat publik (seperti `GET /api/pendaftar/{nomor}` yang sudah ada), endpoint admin di-protect Sanctum.

## 3.3 Data yang Sensitif

| Data | Perlakuan | Alasan |
|------|-----------|--------|
| `pendaftars.email`, `pendaftars.nomor_hp` | Tidak ditampilkan di endpoint publik "Jadwal Saya"; hanya untuk admin & pengiriman notifikasi | Data kontak pribadi (PII), rawan disalahgunakan/spam. |
| Endpoint admin & operator (assign, approve, rekap kehadiran) | Wajib Sanctum token + cek `role` | Mencegah peserta mengubah jadwal/kehadiran orang lain. |
| Token Sanctum operator/admin | Disimpan di `sessionStorage`, tidak di-log | Mencegah pencurian sesi. |
| `permintaan_reschedule.alasan` | Hanya terlihat oleh admin & peserta ybs | Bisa memuat info pribadi (sakit, keluarga). |
| Endpoint "Jadwal Saya" | Hanya kembalikan field jadwal + nama (tanpa email/HP/skor) | Minimal exposure — siapa pun yang tahu nomor pendaftaran hanya melihat jadwal, bukan data sensitif. |

---

# BAGIAN 4 — ERD / Desain Database

## 4.1 Daftar Tabel Baru

| Nama Tabel | Deskripsi |
|------------|-----------|
| `sesi_tes` | Slot/sesi tes yang dibuat admin (jenis, waktu, ruang, kuota). |
| `jadwal_peserta` | Penugasan satu peserta (`pendaftar`) ke satu `sesi_tes` — tabel penghubung utama ke sistem existing. |
| `absensi` | Catatan kehadiran peserta per jadwal, diisi operator di hari-H. |
| `permintaan_reschedule` | Pengajuan perubahan jadwal oleh peserta beserta status persetujuan. |

Tabel existing `pendaftars` dan `users` **direferensikan**, tidak dibuat ulang. (Catatan: `users` diberi tambahan kolom `role` via migration `add_role_to_users_table` — nullable default `admin` agar data lama valid.)

## 4.2 Struktur Tiap Tabel

### `sesi_tes`
| Nama Kolom | Tipe Data | Constraint | Keterangan |
|-----------|-----------|-----------|------------|
| id | BIGINT | PK, AUTO_INCREMENT | — |
| jenis_tes | VARCHAR(20) | NOT NULL | `Tes Tulis` / `Wawancara` (konstanta di model) |
| tanggal | DATE | NOT NULL | Tanggal pelaksanaan |
| jam_mulai | TIME | NOT NULL | — |
| jam_selesai | TIME | NOT NULL | — |
| ruang | VARCHAR(50) | NOT NULL | Lokasi/ruang |
| kuota | SMALLINT | NOT NULL, default 0 | Kapasitas maksimum peserta |
| terisi | SMALLINT | NOT NULL, default 0 | Counter peserta ter-assign (denormalisasi untuk cek kuota cepat) |
| status | VARCHAR(15) | NOT NULL, default `aktif` | `aktif` / `penuh` / `selesai` / `dibatalkan` |
| created_at, updated_at | TIMESTAMP | NULL | timestamps() |

### `jadwal_peserta`
| Nama Kolom | Tipe Data | Constraint | Keterangan |
|-----------|-----------|-----------|------------|
| id | BIGINT | PK, AUTO_INCREMENT | — |
| pendaftar_id | BIGINT | FK → pendaftars.id, NOT NULL, ON DELETE CASCADE | Titik integrasi ke sistem existing |
| sesi_tes_id | BIGINT | FK → sesi_tes.id, NOT NULL | — |
| jenis_tes | VARCHAR(20) | NOT NULL | Disalin dari sesi untuk constraint unik per jenis |
| status_kehadiran | VARCHAR(15) | NOT NULL, default `Terjadwal` | `Terjadwal` / `Hadir` / `Tidak Hadir` |
| created_at, updated_at | TIMESTAMP | NULL | — |
| | | **UNIQUE(pendaftar_id, jenis_tes)** | Cegah jadwal ganda untuk jenis tes yang sama |

### `absensi`
| Nama Kolom | Tipe Data | Constraint | Keterangan |
|-----------|-----------|-----------|------------|
| id | BIGINT | PK, AUTO_INCREMENT | — |
| jadwal_peserta_id | BIGINT | FK → jadwal_peserta.id, NOT NULL, ON DELETE CASCADE | — |
| dicatat_oleh | BIGINT | FK → users.id, NOT NULL | Operator yang mencatat |
| status | VARCHAR(15) | NOT NULL | `Hadir` / `Tidak Hadir` |
| waktu_checkin | TIMESTAMP | NULL | Saat operator menandai hadir |
| created_at, updated_at | TIMESTAMP | NULL | — |

### `permintaan_reschedule`
| Nama Kolom | Tipe Data | Constraint | Keterangan |
|-----------|-----------|-----------|------------|
| id | BIGINT | PK, AUTO_INCREMENT | — |
| jadwal_peserta_id | BIGINT | FK → jadwal_peserta.id, NOT NULL | Jadwal yang ingin diubah |
| sesi_tujuan_id | BIGINT | FK → sesi_tes.id, NOT NULL | Sesi alternatif yang dipilih peserta |
| alasan | TEXT | NOT NULL | Alasan reschedule |
| status | VARCHAR(15) | NOT NULL, default `Diajukan` | `Diajukan` / `Disetujui` / `Ditolak` |
| catatan_admin | VARCHAR(255) | NULL | Alasan admin jika menolak |
| diproses_oleh | BIGINT | FK → users.id, NULL | Admin yang memproses |
| created_at, updated_at | TIMESTAMP | NULL | — |

## 4.3 Relasi Antar Tabel

```
[pendaftars] ---(1 : N)--- [jadwal_peserta]
Keterangan: Satu pendaftar (yang Lolos Seleksi) bisa punya beberapa jadwal (mis. 1 Tes Tulis + 1 Wawancara).
            FK jadwal_peserta.pendaftar_id → pendaftars.id. Inilah jembatan ke sistem existing.

[sesi_tes] ---(1 : N)--- [jadwal_peserta]
Keterangan: Satu sesi menampung banyak peserta hingga batas kuota. FK jadwal_peserta.sesi_tes_id → sesi_tes.id.

[jadwal_peserta] ---(1 : 1)--- [absensi]
Keterangan: Tiap jadwal punya paling banyak satu catatan kehadiran, diisi operator saat hari-H.

[jadwal_peserta] ---(1 : N)--- [permintaan_reschedule]
Keterangan: Satu jadwal bisa diajukan reschedule (bisa lebih dari sekali jika ditolak lalu diajukan lagi).

[sesi_tes] ---(1 : N)--- [permintaan_reschedule]
Keterangan: permintaan_reschedule.sesi_tujuan_id → sesi_tes.id (sesi alternatif yang dituju).

[users] ---(1 : N)--- [absensi]   &   [users] ---(1 : N)--- [permintaan_reschedule]
Keterangan: Mencatat siapa (operator/admin) yang melakukan aksi — audit trail. FK ke users.id existing.
```

## 4.4 Indexing

| Tabel.Kolom | Jenis | Alasan |
|-------------|-------|--------|
| `jadwal_peserta.pendaftar_id` | Index (FK) | Query "Jadwal Saya" sering filter by pendaftar (JOIN pendaftars). |
| `jadwal_peserta.sesi_tes_id` | Index (FK) | Admin sering ambil semua peserta dalam satu sesi + hitung `terisi`. |
| `jadwal_peserta (pendaftar_id, jenis_tes)` | UNIQUE composite | Menegakkan aturan "1 jadwal per jenis tes" sekaligus mempercepat lookup. |
| `sesi_tes.tanggal` | Index | Filter sesi berdasarkan tanggal (mis. jadwal hari ini untuk operator). |
| `sesi_tes.status` | Index | Sering query `WHERE status='aktif'` saat menampilkan sesi yang bisa dipilih. |
| `permintaan_reschedule.status` | Index | Admin filter permintaan `Diajukan` di inbox-nya. |

Index difokuskan ke kolom FK dan kolom yang dipakai di klausa `WHERE`/`JOIN` paling sering — bukan semua kolom — untuk menghindari overhead tulis yang tidak perlu.

---

# BAGIAN 5 — Prompt Siap Pakai untuk AI

> Prompt tunggal di bawah ini merangkum keputusan Bagian 1–4 dan dirancang self-contained agar AI melanjutkan sistem existing, bukan membangun dari nol. Menggunakan formula 5 komponen.

```
[KONTEKS]
Saya sedang mengembangkan aplikasi PMB (Penerimaan Mahasiswa Baru) yang SUDAH BERJALAN.
Stack: React 18 + Vite + Tailwind CSS (frontend, routing via window tanpa react-router,
HTTP pakai Fetch API lewat utils/api.js, token admin di sessionStorage) dan Laravel 12 +
Sanctum + SQLite (backend). Fitur existing yang TIDAK BOLEH diubah/dirusak: pendaftaran
online, cek status, dashboard admin + statistik, update status, login Sanctum, export CSV,
heregistrasi. Tabel existing: `pendaftars` (id, nomor_pendaftaran, nama, nomor_hp, email,
asal_sekolah, prodi, jalur, status, heregistrasi_at) dengan status 'Menunggu'/'Lolos Seleksi'/
'Tidak Lolos', dan `users` (admin). Response API SELALU format {success, message, data} /
{success, errors}. Ikuti konvensi di skill.md (model PascalCase singular, route kebab-case,
enum sebagai VARCHAR + konstanta di model, komponen functional + Tailwind only).

[TUJUAN]
Tambahkan MODUL PENJADWALAN TES SELEKSI & WAWANCARA sebagai fitur baru DI ATAS sistem ini.
Tujuannya mendigitalkan distribusi jadwal: peserta yang berstatus 'Lolos Seleksi' mendapat
sesi tes, bisa melihat & download kartu tes, mengajukan reschedule, dan operator mencatat
kehadiran di hari-H. Jangan reset arsitektur; perluas saja.

[FITUR]
Backend (Laravel) — buat 4 tabel baru via migration: sesi_tes, jadwal_peserta (FK pendaftar_id
→ pendaftars.id, UNIQUE(pendaftar_id, jenis_tes)), absensi (FK jadwal_peserta_id, dicatat_oleh →
users.id), permintaan_reschedule. Tambah kolom `role` (nullable default 'admin') ke users.
Endpoint baru:
  - POST   /api/jadwal/sesi                         (admin) buat sesi
  - GET    /api/jadwal/sesi                          (admin) list sesi
  - POST   /api/jadwal/assign                        (admin) assign peserta Lolos Seleksi ke sesi (cek kuota)
  - GET    /api/jadwal/peserta/{nomorPendaftaran}    (publik) lihat jadwal + kartu tes
  - POST   /api/jadwal/reschedule                    (publik) peserta ajukan reschedule
  - PATCH  /api/jadwal/reschedule/{id}               (admin) approve/reject
  - POST   /api/jadwal/{jadwalPesertaId}/absensi     (operator) catat kehadiran
  - GET    /api/jadwal/sesi/{id}/peserta             (operator/admin) daftar peserta per sesi
Frontend (React) — komponen baru di src/components/pmb/: FormSesiTes.jsx, DaftarSesi.jsx,
AssignPeserta.jsx, JadwalSaya.jsx (input nomor pendaftaran → kartu tes + QR via qrcode.react),
FormReschedule.jsx, AbsensiOperator.jsx. Tambah tab "Jadwal" di dashboard admin dan menu publik
"Jadwal Saya" di Home. Reuse pola TabelPendaftar.jsx & komponen ui/ existing.

[CONSTRAINT]
- JANGAN ubah skema tabel pendaftars; integrasi hanya lewat FK pendaftar_id.
- Hanya peserta status 'Lolos Seleksi' yang boleh di-assign (validasi server, tolak 422 jika tidak).
- Tolak assign jika kuota penuh (422) dan jika peserta sudah punya jadwal jenis tes sama (409).
- Endpoint admin/operator di-protect middleware auth:sanctum; bedakan operator vs admin lewat users.role.
- Endpoint "Jadwal Saya" publik hanya kembalikan nama + detail jadwal, JANGAN expose email/nomor_hp.
- Pakai Laravel Notification (mail driver 'log' di dev) + Queue (driver database) untuk notifikasi/reminder.
- Semua response ikut format {success, message, data}. Index kolom FK + sesi_tes.tanggal/status.

[TAMPILAN]
Konsisten dengan UI existing: Tailwind only, mobile-first (min 375px), warna biru utama #1a56db
(blue-600/700), aksen amber-500, teks slate-800/500. Status badge ikut pola StatusBadge.jsx:
Terjadwal=blue-100/blue-800, Hadir=green-100/green-800, Tidak Hadir=red-100/red-800. Tombol min
tinggi 44px. Kartu tes berupa card rapi berisi nomor pendaftaran, nama, jenis tes, tanggal, jam,
ruang, dan QR code. Tabel admin punya filter real-time seperti TabelPendaftar.jsx.

Kerjakan bertahap: 1) migration + model + relasi, 2) controller + routes + validasi,
3) komponen frontend + integrasi Fetch, 4) notifikasi. Setelah tiap tahap, pastikan fitur lama
tetap berjalan.
```

---

# BAGIAN 6 — Jalankan Prompt & Evaluasi Hasil (Bonus)

> Bagian ini diisi **setelah** prompt Bagian 5 dijalankan dan modul benar-benar berjalan di browser. Di bawah adalah kerangka log + tabel evaluasi yang akan diisi saat eksekusi.

## 6.1 Log Prompt & Iterasi

**Prompt Utama:** (lihat Bagian 5 — dikirim ke AI sebagai instruksi awal)

**Iterasi 1 — _alasan:_** _(diisi: apa yang kurang/tidak sesuai dari hasil pertama, mis. validasi kuota belum atomic / kartu tes belum tampil QR)_
```
(prompt lanjutan ditulis di sini)
```

**Iterasi 2 — _alasan:_** _(diisi bila perlu)_

## 6.2 Tabel Evaluasi Kesesuaian dengan Plan

| Item dari Plan | Bagian | Status | Catatan |
|----------------|--------|--------|---------|
| CRUD sesi tes (admin) | 1.2 | ⬜ | _diisi setelah run_ |
| Assign peserta Lolos Seleksi + cek kuota | 1.2 / 2.1 | ⬜ | |
| Halaman "Jadwal Saya" + kartu tes | 1.2 / 3.2 | ⬜ | |
| Permintaan reschedule (flow 2.2) | 2.2 | ⬜ | |
| Catat kehadiran operator | 1.2 | ⬜ | |
| Tabel `sesi_tes` | 4.2 | ⬜ | |
| Tabel `jadwal_peserta` (+UNIQUE) | 4.2 | ⬜ | |
| Tabel `absensi` | 4.2 | ⬜ | |
| Tabel `permintaan_reschedule` | 4.2 | ⬜ | |
| **Regresi:** pendaftaran masih jalan | — | ⬜ | _wajib dicek_ |
| **Regresi:** cek status masih jalan | — | ⬜ | |
| **Regresi:** dashboard admin + login masih jalan | — | ⬜ | |

Legenda: ✅ sesuai · ⚠️ sebagian / beda detail · ❌ tidak terpenuhi.

## 6.3 Struktur Repo Akhir (target)

```
admission-app/
├── devplan.md                 ← file ini (plan + log prompt + evaluasi)
└── app/
    ├── pmb-frontend/          ← extend dari sistem existing
    ├── pmb-backend/           ← extend dari sistem existing
    └── README-app.md          ← cara menjalankan + fitur baru + konfirmasi tanpa regresi
```

---

*Disusun untuk Vibe Coding & Venture SEVIMA — pengembangan lanjutan aplikasi PMB.*
