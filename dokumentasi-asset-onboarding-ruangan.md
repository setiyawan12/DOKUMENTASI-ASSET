# Dokumentasi Teknis & Manual Book — Alokasi Asset ke Ruangan / Onboarding Ruangan (ALK-RGN01)

> Dokumen ini disusun **dari source code lokal** (`API/` Lumen dan `MOBILE/` Flutter) per 13 Oktober 2026, dibandingkan dengan dokumen bisnis **"2.1 Ruangan - Alokasi Aset ke Ruangan (Onboarding) (ALK-RGN01)" v1.0**.
> Cakupan implementasi: alokasi **aset gudang berstatus `tersedia`** ke **ruangan aktif**, sisi **Admin/HR** dan **PIC Ruangan**. Bagian yang tidak ditemukan ditandai **⚠️ Belum ditemukan**, asumsi ditandai **⚠️ Asumsi**.
> Contoh ID/nama pada payload adalah **data sintetis**.

---

## 1. Informasi Dokumen

| Informasi | Nilai |
|---|---|
| Nama Fitur | Alokasi Asset ke Ruangan (UI Admin: **"On Boarding"** di dashboard Ruangan; UI PIC: kartu **"PIC Ruangan"** di halaman pertama Asset) |
| Kode Fitur | **ALK-RGN01** (nomor transaksi `RGN01-YYYYMMNNN`) |
| Modul | Asset Management — Ruangan |
| Backend | Laravel **Lumen ^10.0**, PHP ^8.1, JWT, Query Builder |
| Database | `API/diagram-asset.dbml` (tanda `[NEW ALK-RGN01]` / `[REVISI ALK-RGN01]`) + SQL manual `Claude outputs/sql-alk-rgn01-onboarding-ruangan.sql` (sudah dijalankan di environment pengembang) |
| Mobile | Flutter (`professional_flutter_3_35`), `flutter_bloc`, `dio`, `dotted_border` |
| Web | ⚠️ Belum ditemukan: tidak ada source web di workspace |
| Test otomatis | API `tests/Asset/AssetAdminRuanganTest.php` (10 test) + `tests/Asset/AssetLokasiPicTest.php` (7 test) · Mobile `test/asset/asset_admin_ruangan_test.dart` (9 test) + `test/asset/asset_pic_ruangan_test.dart` (9 test) |
| Versi dokumen | 1.1 — 22 Oktober 2026 |
| Dokumen terkait | Onboarding Karyawan (KRY01), Master Lokasi & PIC Ruangan |

## 2. Daftar Isi

Lihat panel navigasi di samping.

## 3. Quick Overview

Admin/HR memilih satu atau lebih **aset gudang**, **ruangan tujuan**, dan **jadwal serah terima**. PIC Ruangan ditentukan dari master (`asset_lokasi_pic_m`) dan **terkunci**; hanya ruangan tanpa PIC yang boleh dipilihkan karyawan, dan pilihan itu **disimpan ke master**. Aset berubah `tersedia` → `proses` sehingga tidak bisa dialokasikan ganda. PIC menerima aset secara fisik, menilai **setiap aset Normal atau Rusak** (keterangan wajib bila rusak), lalu mengirim checklist. Admin/HR mereview: bila semua normal ia **mengonfirmasi** (aset menjadi `terpakai` di ruangan); bila ada yang rusak ia **menjadwalkan pengganti** dari gudang dan serah terima diulang pada **ronde** berikutnya sampai semua normal.

| Aspek | Ringkasan |
|---|---|
| Aktor | Admin/HR (Admin Asset), PIC Ruangan, Sistem |
| Fase (diturunkan dari data) | `menunggu_serah_terima` → `menunggu_review` → `selesai` (berulang per ronde bila ada aset rusak) |
| Status header | `status_approval`: `proses` → `dikonfirmasi` (enum sama dengan KRY01; `draft` dan `ditolak` tidak dipakai di alur ini) |
| Status asset | `tersedia` → `proses` → `terpakai` (pemegang `ruangan`); aset rusak → `rusak` (kembali ke gudang) |
| Endpoint | 7 admin + 3 PIC |
| Notifikasi | 5 event (`onboarding_ruangan.*`) |

## 4. Ringkasan Fitur

| No | Fitur | Aktor |
|---|---|---|
| 1 | Buat alokasi: pilih ruangan, asset (pemilih kategori → asset), jadwal, catatan | Admin/HR |
| 2 | PIC otomatis dari master ruangan; ruangan tanpa PIC → pilih karyawan dan simpan ke master | Admin/HR |
| 3 | Ubah jadwal serah terima sebelum PIC menerima (riwayat jadwal tercatat) | Admin/HR |
| 4 | Terima asset dan checklist kondisi per aset | PIC Ruangan |
| 5 | Review hasil checklist, tampilan timeline per aset | Admin/HR |
| 6 | Jadwalkan asset pengganti untuk aset rusak (ronde berikutnya) | Admin/HR |
| 7 | Konfirmasi selesai: aset menjadi `terpakai` di ruangan | Admin/HR |
| 8 | Daftar asset ruangan milik PIC (halaman "PIC Ruangan") | PIC Ruangan |

## 5. Tujuan Fitur

### 5.1 Tujuan Bisnis (diturunkan dari dokumen bisnis dan kode)

- Melengkapi ruangan (mis. ruangan baru) dengan aset dari gudang secara terdokumentasi.
- Penanggung jawab ruangan (PIC) menerima dan memeriksa aset secara fisik sebelum aset resmi tercatat di ruangan.
- Aset rusak tidak pernah masuk ruangan: wajib diganti sebelum proses selesai.
- Jejak lengkap: siapa, kapan, jadwal berapa kali berubah, ronde penggantian.

### 5.2 Tujuan Sistem

- Mencegah alokasi ganda (`proses`) dan onboarding ganda per ruangan.
- Menurunkan fase dari data (`tgl_terima_pic` pada jadwal) tanpa kolom status tambahan.
- Satu transaksi mencakup banyak ronde (pengganti) tanpa membuat transaksi baru.

## 6. Aktor

| Aktor | Peran |
|---|---|
| Admin/HR | Pengguna terdaftar di `asset_admin`. Membuat alokasi, mengubah jadwal, menjadwalkan pengganti, mengonfirmasi |
| PIC Ruangan | Karyawan yang melekat pada ruangan (`asset_lokasi_pic_m`) atau dipilih Admin/HR untuk ruangan tanpa PIC. Menerima aset dan mengirim checklist |
| Sistem | Mengubah status aset, mencatat log, mengirim notifikasi |

## 7. Hak Akses

| Aksi | Admin/HR | PIC Ruangan | Lainnya |
|---|---|---|---|
| Daftar/detail/buat/ubah jadwal/pengganti/konfirmasi | Ya | Tidak | Tidak (`FORBIDDEN`) |
| Daftar ruangan & aset milik sendiri, detail serah terima, terima | Tidak | Hanya transaksi dengan `id_karyawan_pic` = dirinya | Tidak |

Cek admin lewat `accessError` (tabel `asset_admin`). PIC dicek pada setiap query (`id_karyawan_pic` = karyawan login), jadi PIC lain mendapat `NOT_FOUND`.

## 8. Terminologi

| Istilah | Arti |
|---|---|
| Onboarding ruangan | Alokasi aset gudang ke ruangan tujuan (kebalikan: offboarding ruangan ALK-RGN02) |
| Ronde | Putaran serah terima; ronde 1 = aset awal, ronde berikutnya = unit pengganti |
| Rantai penggantian | `parent_id` pada baris pengganti = baris aset rusak yang digantikannya (sama dengan KRY01) |
| Master PIC | `asset_lokasi_pic_m`: satu ruangan satu PIC |
| Fase | Tahap serah terima, diturunkan dari data, bukan kolom |

## 9. Business Rules

### BR-001 — Aset yang boleh dialokasikan
Hanya `status_asset = 'tersedia'` dan `pemegang_asset = 'gudang'` (`getLockedAssets`). Pesan: "Satu atau beberapa asset tidak tersedia untuk dialokasikan."

### BR-002 — Ruangan tujuan
`asset_lokasi_m.tipe = 'ruangan'` dan `status_aktif = '1'`. Pesan: "Ruangan tujuan tidak ditemukan atau tidak aktif."

### BR-003 — Satu ruangan satu kali onboarding
Ditolak bila ruangan sudah punya transaksi berstatus `proses` atau `dikonfirmasi`. Pesan: "Ruangan ini sudah pernah di-onboarding. Onboarding hanya dapat dilakukan satu kali per ruangan." Ruangan tersebut juga disaring dari `options`.

### BR-004 — PIC Ruangan
PIC dari `asset_lokasi_pic_m` otomatis dipakai dan **tidak dapat diganti** ("PIC Ruangan sudah ditetapkan pada ruangan ini dan tidak dapat diganti."). Ruangan tanpa PIC: Admin/HR wajib memilih karyawan aktif; pilihan itu **disimpan ke `asset_lokasi_pic_m`** dalam transaksi yang sama, dan `id_asset_lokasi_pic_m` header menunjuk baris baru itu.

### BR-005 — Jadwal serah terima
Tanggal (`Y-m-d`) dan jam (`H:i`) wajib; tidak boleh sebelum hari ini ("Tanggal serah terima tidak boleh sebelum tanggal pengajuan."). Header menyimpan jadwal terkini; riwayat di `asset_trx_onboarding_ruangan_jadwal`.

### BR-006 — Ubah jadwal
Hanya selama fase `menunggu_serah_terima` (PIC belum menerima). Menambah baris jadwal pada ronde yang sama. Pesan: "Jadwal hanya dapat diubah sebelum PIC Ruangan menerima asset."

### BR-007 — Checklist PIC
Wajib mengisi **semua** aset ronde berjalan (Normal/Rusak). Rusak wajib keterangan ("Keterangan kerusakan wajib diisi."). Pesan bila kurang: "Seluruh item pada serah terima wajib dicek." PIC **tidak mengubah status aset**; menunggu keputusan Admin/HR.

### BR-008 — Penggantian
Hanya di fase `menunggu_review`. **Semua** aset rusak ronde ini wajib diberi pengganti ("Seluruh asset rusak wajib diberi asset pengganti."), pengganti harus `tersedia` di gudang dan tidak boleh sama dua kali. Aset rusak → `status_asset = 'rusak'` (`ket_rusak` terisi, tetap di gudang); pengganti → `proses`. Baris pengganti: `ronde` + 1, `parent_id` = baris rusak. Jadwal baru wajib.

### BR-009 — Konfirmasi selesai
Hanya di fase `menunggu_review` dan **tanpa aset rusak**. Semua aset aktif (belum digantikan) menjadi `terpakai`, `pemegang_asset = 'ruangan'`, `id_pemegang_asset` dan `id_asset_lokasi_m` = id ruangan; `asset_trx_log` jenis `onboarding_ruangan` dicatat per aset. Header → `dikonfirmasi` (`user_approval`, `tgl_approval`).

### BR-010 — Penomoran
`id` = `id_company` + `YYYYMM` + 3 digit; `no_trx` = `RGN01-YYYYMMNNN`. Maksimal 999 transaksi per bulan per company ("Nomor transaksi untuk periode ini sudah mencapai batas.").

### BR-011 — Skema harus sudah diterapkan
Fitur butuh SQL manual `sql-alk-rgn01-onboarding-ruangan.sql` (kolom baru, tabel jadwal, unique PIC).

## 10. Business Flow

1. Admin/HR membuka dashboard Ruangan → **On Boarding** → **Tambah Alokasi**.
2. Memilih ruangan (PIC otomatis terisi/terkunci, atau dipilih bila belum ada), asset, jadwal, catatan, lalu **Kirim ke PIC Ruangan**. Aset → `proses`; PIC dinotifikasi.
3. Pada jadwal, Admin/HR menyerahkan aset secara fisik; PIC menilai tiap aset dan mengirim checklist. Admin dinotifikasi.
4. Admin/HR mereview di detail (timeline per aset).
   - **Semua normal** → **Konfirmasi Selesai**: aset `terpakai` di ruangan.
   - **Ada rusak** → **Jadwalkan Pengganti**: pilih pengganti per aset rusak + jadwal baru; PIC dinotifikasi; kembali ke langkah 3 pada ronde berikutnya.
5. Selama PIC belum menerima, Admin/HR bisa **Ubah Jadwal**; PIC dinotifikasi.

## 11. Flow Berdasarkan Role

### 11.1 Admin/HR
Buat alokasi → (ubah jadwal) → tunggu PIC → review → konfirmasi atau jadwalkan pengganti.

### 11.2 PIC Ruangan
Terima notifikasi → buka banner pada halaman "PIC Ruangan" → periksa fisik, nilai tiap aset → kirim → (tunggu review; bila ada pengganti, ulangi untuk unit pengganti).

### 11.3 System
Kunci aset (`proses`), turunkan fase, catat jadwal/log, kirim notifikasi, simpan PIC ke master.

## 12. Flow Diagram

```mermaid
flowchart TD
  A[Admin/HR buat alokasi] --> B[Aset tersedia → proses<br/>PIC dinotifikasi]
  B --> C{Jadwal berubah?}
  C -- ya, PIC belum menerima --> D[Ubah jadwal + notifikasi] --> B
  C -- tidak --> E[PIC terima aset &<br/>kirim checklist per aset]
  E --> F{Ada aset rusak?}
  F -- tidak --> G[Admin/HR konfirmasi]
  G --> H[Aset terpakai di ruangan<br/>+ asset_trx_log]
  F -- ya --> I[Admin/HR pilih pengganti<br/>+ jadwal baru]
  I --> J[Aset rusak → rusak di gudang<br/>pengganti → proses, ronde + 1]
  J --> E
```

## 13. Sequence Diagram

```mermaid
sequenceDiagram
  participant A as Admin/HR (mobile)
  participant API as API Lumen
  participant DB as Database
  participant P as PIC (mobile)
  A->>API: POST asset/onboarding-ruangan
  API->>DB: header + item (ronde 1) + jadwal; asset_data → proses
  API-->>P: notifikasi onboarding_ruangan.dikirim
  P->>API: POST asset/karyawan/pic-ruangan/onboarding/{id}/terima
  API->>DB: item.status_item, jadwal.tgl_terima_pic
  API-->>A: notifikasi onboarding_ruangan.diterima
  alt semua normal
    A->>API: POST …/{id}/konfirmasi
    API->>DB: asset_data → terpakai/ruangan; asset_trx_log; header dikonfirmasi
  else ada rusak
    A->>API: POST …/{id}/pengganti
    API->>DB: aset rusak → rusak; pengganti → proses; item ronde+1; jadwal baru
    API-->>P: notifikasi onboarding_ruangan.pengganti
  end
```

## 14. Status Transaksi

| Fase (turunan) | Syarat | Tampilan |
|---|---|---|
| `menunggu_serah_terima` | `status_approval='proses'` dan jadwal ronde terakhir `tgl_terima_pic` kosong | Menunggu Serah Terima |
| `menunggu_review` | `proses` dan `tgl_terima_pic` terisi | Menunggu Review Admin |
| `selesai` | `status_approval='dikonfirmasi'` | Selesai |

## 15. State Transition

| Dari | Aksi | Ke |
|---|---|---|
| (baru) | `store` | `menunggu_serah_terima` (ronde 1) |
| `menunggu_serah_terima` | `updateJadwal` | `menunggu_serah_terima` (baris jadwal baru) |
| `menunggu_serah_terima` | PIC `terima` | `menunggu_review` |
| `menunggu_review` | `jadwalkanPengganti` | `menunggu_serah_terima` (ronde + 1) |
| `menunggu_review` | `konfirmasi` (tanpa rusak) | `selesai` |

## 16. Struktur Database

| Tabel | Perubahan | Fungsi |
|---|---|---|
| `asset_trx_onboarding_ruangan` | header (status enum `status_onboarding`, `id_karyawan_pic`, jadwal boleh NULL) | Transaksi |
| `asset_trx_onboarding_ruangan_item` | `+ ronde`; `parent_id` = rantai pengganti | Satu baris per aset |
| `asset_trx_onboarding_ruangan_jadwal` | **tabel baru** | Riwayat jadwal per ronde |
| `asset_lokasi_pic_m` | unique `(id_company, id_asset_lokasi_m)` | 1 ruangan = 1 PIC |
| `asset_lokasi_m` | `+ status_aktif` | Hanya ruangan aktif |
| `asset_data` | status/pemegang/lokasi diubah | Dampak aset |
| `asset_trx_log` | jenis `onboarding_ruangan` | Jejak |

## 17. Relasi Database

```mermaid
erDiagram
  asset_trx_onboarding_ruangan ||--o{ asset_trx_onboarding_ruangan_item : memiliki
  asset_trx_onboarding_ruangan ||--o{ asset_trx_onboarding_ruangan_jadwal : riwayat
  asset_trx_onboarding_ruangan_item }o--|| asset_data : aset
  asset_trx_onboarding_ruangan_item }o--o| asset_trx_onboarding_ruangan_item : "parent_id (digantikan)"
  asset_trx_onboarding_ruangan }o--|| asset_lokasi_m : ruangan
  asset_trx_onboarding_ruangan }o--o| asset_lokasi_pic_m : "id_asset_lokasi_pic_m"
  asset_lokasi_pic_m }o--|| asset_lokasi_m : ruangan
```

## 18. Data Dictionary

### 18.1 asset_trx_onboarding_ruangan

| Kolom | Tipe | Catatan |
|---|---|---|
| `id_asset_trx_onboarding_ruangan` | varchar(50) PK | `id_company`+`YYYYMM`+3 digit |
| `no_trx` | varchar(50) | `RGN01-YYYYMMNNN` |
| `tgl_onboarding` | date | Tanggal dibuat |
| `id_asset_lokasi_m`, `nama_asset_ruangan`, `nama_cabang`, `nama_departemen` | | Ruangan + snapshot |
| `id_asset_lokasi_pic_m` | varchar(50) NULL | Baris master PIC |
| `id_karyawan_pic`, `nama_pic`, `nik_pic` | | PIC penerima + snapshot |
| `tgl_serah_terima`, `jam_serah_terima` | date, time NULL | Jadwal **terkini** |
| `keterangan_serah_terima` | varchar(255) | Catatan untuk PIC |
| `keterangan`, `ket_tolak` | text, varchar | Cadangan (dari pola KRY01), tidak dipakai alur ini |
| `id_karyawan`, `nik_karyawan`, `nama_karyawan` | | Admin/HR pembuat |
| `status_approval` | enum | `proses` / `dikonfirmasi` dipakai; `draft`, `ditolak` tidak |
| `user_approval`, `tgl_approval` | | Pelaku & waktu konfirmasi |
| `id_company`, `tgl_input`, `user_input`, `tgl_update`, `user_update` | | Audit |

### 18.2 asset_trx_onboarding_ruangan_item

| Kolom | Tipe | Catatan |
|---|---|---|
| `id_asset_trx_onboarding_ruangan_item` | varchar(50) PK | |
| `id_asset_trx_onboarding_ruangan` | FK | |
| `parent_id` | varchar(50), default `''` | Baris pengganti: id baris aset rusak yang digantikan |
| `id_asset_data` | FK | |
| `nama_asset`, `deskripsi_asset`, `kategori_asset` | | Snapshot |
| `ronde` | tinyint | 1 = awal; pengganti = ronde berikutnya |
| `status_item` | enum `kondisi_awal` | `belum_dicek`, `normal`, `rusak` |
| `ket_rusak` | varchar(255) | Wajib bila rusak |

### 18.3 asset_trx_onboarding_ruangan_jadwal

| Kolom | Tipe | Catatan |
|---|---|---|
| `id_asset_trx_onboarding_ruangan_jadwal` | varchar(50) PK | |
| `id_asset_trx_onboarding_ruangan` | FK | |
| `ronde` | tinyint | |
| `tanggal`, `jam` | date, time | |
| `tgl_terima_pic` | timestamp NULL | Terisi saat PIC mengirim checklist |
| `keterangan` | varchar(255) | Alasan perubahan |

Append-only: baris terakhir per ronde = jadwal aktif ronde itu.

## 19. Database Index

| Index | Kolom |
|---|---|
| `idx_onb_rgn_status` | `(id_company, status_approval, tgl_approval, tgl_input, id)` |
| `idx_onb_rgn_pic` | `(id_company, id_karyawan_pic, status_approval)` |
| `idx_onb_rgn_item_trx` | `(id_trx, id_company, status_item, parent_id)` |
| `idx_onb_rgn_item_asset` | `(id_asset_data, id_company)` |
| `idx_onb_rgn_jadwal_trx` | `(id_trx, ronde, id_company)` |
| `uq_asset_ruangan_pic_lokasi` | UNIQUE `(id_company, id_asset_lokasi_m)` |

## 20. Backend Architecture

| Lapisan | File |
|---|---|
| Route | `API/routes/modules/asset/RouteAsset.php` (grup `onboarding-ruangan`, `karyawan/pic-ruangan`) |
| Controller Admin | `Controllers/Asset/Admin/AssetAdminOnboardingRuangan_c.php` |
| Controller PIC | `Controllers/Asset/Karyawan/AssetKaryawanPicRuangan_c.php` |
| Model Admin | `Models/Asset/AssetAdminRuangan_m.php` |
| Model PIC + helper bersama | `Models/Asset/AssetLokasiPic_m.php` (`fase`, `rondeSaatIni`, `jadwalTerakhir`, `itemChecklist`, `buildDetail`) |
| Notifikasi | `Support/Asset/AssetNotification.php` |

Setiap mutasi memakai `DB::transaction`, `lockCompany` (kunci per company), dan penomoran `nextSequence`.

## 21. Backend Business Logic

### 21.1 Buat — `store`
Validasi admin → ruangan aktif → cek sekali-onboarding → resolve PIC (master terkunci / pilihan manual) → kunci aset (`getLockedAssets`) → validasi jadwal → (PIC manual) simpan ke `asset_lokasi_pic_m` → insert header, item ronde 1, jadwal ronde 1 → aset `proses`.

### 21.2 Ubah jadwal — `updateJadwal`
Header `proses`, fase `menunggu_serah_terima` → tambah baris jadwal (ronde sama) → update jadwal terkini header.

### 21.3 Terima (PIC) — `terimaOnboarding`
Header milik PIC login, fase `menunggu_serah_terima` → validasi semua item ronde dicek → update `status_item`/`ket_rusak` → isi `tgl_terima_pic` pada jadwal ronde → `tgl_update` header.

### 21.4 Pengganti — `jadwalkanPengganti`
Fase `menunggu_review` → `asetRusakRondeIni` → cocokkan penuh dengan pilihan admin → aset rusak: `rusak` + `ket_rusak`; pengganti dikunci; insert item ronde+1 (`parent_id` = baris rusak) + jadwal ronde+1 → pengganti `proses`.

### 21.5 Konfirmasi — `konfirmasi`
Fase `menunggu_review`, tanpa rusak → semua aset aktif divalidasi `proses`/`gudang` → `terpakai`/`ruangan` + lokasi → `asset_trx_log` → header `dikonfirmasi`.

### 21.6 Query baca
`getOptions` (aset tersedia + pohon kategori, ruangan aktif belum onboarding beserta PIC, karyawan aktif), `getPage` (tab `proses`/`selesai`, paginasi), `getDetail` (+ `bisa_ubah_jadwal`, `bisa_konfirmasi`, `perlu_pengganti`), sisi PIC `getAssetRuanganSaya`, `getDetailOnboardingSaya`.

## 22. Dampak terhadap asset_data

| Peristiwa | `status_asset` | `pemegang_asset` | Lokasi |
|---|---|---|---|
| Buat alokasi | `tersedia` → `proses` | `gudang` | tetap |
| Aset dinilai rusak lalu diganti | `proses` → `rusak` | `gudang` | tetap |
| Pengganti dijadwalkan | `tersedia` → `proses` | `gudang` | tetap |
| Konfirmasi selesai | `proses` → `terpakai` | `ruangan` (`id_pemegang_asset` = id ruangan) | `id_asset_lokasi_m` = ruangan |

## 23. API Documentation

Base: `asset/` · Admin: Bearer + `asset_admin`.

| ID | Method | Path | Fungsi |
|---|---|---|---|
| API-001 | GET | `onboarding-ruangan/options` | Aset tersedia, kategori, ruangan, karyawan |
| API-002 | GET | `onboarding-ruangan?tab=&limit=&offset=` | Daftar (`proses`/`selesai`) |
| API-003 | POST | `onboarding-ruangan` | Buat alokasi |
| API-004 | GET | `onboarding-ruangan/{id}` | Detail |
| API-005 | PUT | `onboarding-ruangan/{id}/jadwal` | Ubah jadwal |
| API-006 | POST | `onboarding-ruangan/{id}/pengganti` | Jadwalkan pengganti |
| API-007 | POST | `onboarding-ruangan/{id}/konfirmasi` | Konfirmasi selesai |
| API-008 | GET | `karyawan/pic-ruangan` | Ruangan & aset PIC + serah terima menunggu |
| API-009 | GET | `karyawan/pic-ruangan/onboarding/{id}` | Detail serah terima PIC |
| API-010 | POST | `karyawan/pic-ruangan/onboarding/{id}/terima` | Kirim checklist |

## 24. Detail Setiap API

### API-003 — Buat
Request:
```json
{ "id_asset_lokasi_m": "RGN-1", "asset_ids": ["AST-1", "AST-2"],
  "id_karyawan_pic": "PIC-2", "tanggal": "2026-10-15", "jam": "09:00", "catatan": "Bawa kunci" }
```
`id_karyawan_pic` hanya dikirim untuk ruangan **tanpa** PIC master. Respons: `{ "id", "no_trx", "status_approval": "proses" }` (`CREATED`).

### API-005 — Ubah jadwal
`{ "tanggal": "2026-10-16", "jam": "13:30", "keterangan": "Ruangan belum siap" }` → `UPDATED`.

### API-006 — Pengganti
```json
{ "items": [{ "id_item": "<id baris rusak>", "id_asset_data": "AST-3" }],
  "tanggal": "2026-10-20", "jam": "10:00", "keterangan": "Unit pengganti" }
```
Respons: `{ "id", "no_trx", "ronde", "jumlah_pengganti" }`.

### API-007 — Konfirmasi
Tanpa body. Respons: `{ "id", "no_trx", "status_approval": "dikonfirmasi", "jumlah_asset" }`.

### API-004 / API-009 — Detail
Field penting: `fase`, `fase_label`, `ronde`, `bisa_diterima` (PIC) / `bisa_ubah_jadwal`, `bisa_konfirmasi`, `perlu_pengganti[]` (admin), `jadwal[]`, `items[]` (`id_item`, `parent_id`, `id_asset_data`, `nama_asset`, `ronde`, `status_item`, `ket_rusak`, `diganti`, `perlu_dicek`).

### API-010 — Terima (PIC)
```json
{ "items": [
  { "id_item": "ITM-1", "status_item": "normal" },
  { "id_item": "ITM-2", "status_item": "rusak", "ket_rusak": "Mati total" } ] }
```
Respons: `{ "id", "no_trx", "ada_rusak", "fase": "menunggu_review" }`.

## 25. Mobile Architecture

| Lapisan | File (`MOBILE/lib/`) |
|---|---|
| Model | `Repository/Model/Asset/mAssetPicRuangan.dart`, `mAssetAdminRuangan.dart` |
| API service | `Repository/Service/assetPicRuanganApi.dart`, `assetAdminRuanganApi.dart` |
| Bloc | `Bloc/Asset/AssetPicRuanganBloc.dart`, `AssetPicRuanganDetailBloc.dart`, `AssetAdminRuanganListBloc.dart`, `AssetAdminRuanganFormBloc.dart`, `AssetAdminRuanganDetailBloc.dart` |
| UI PIC | `UI/Modul/Asset/PicRuangan/` |
| UI Admin | `UI/Modul/Asset/AssetAdmin/AssetAdminRuangan/` |
| Teks | grup `pic_ruangan` di `assets/{id,en}/asset.json` (`MenuAsset.picRuangan`) |
| Navigasi | `Bloc/NavigatorBloc.dart`, `Setting/router.dart` |

## 26. Mobile Screen

### 26.1 PIC — "PIC Ruangan" (`HalamanAssetPicRuangan`)
Tampilan mengikuti Asset Saya: hero (jumlah aset, jumlah ruangan, perlu tindakan), banner serah terima, daftar aset per ruangan.

### 26.2 PIC — Detail serah terima (`HalamanDetailOnboardingPicRuangan`)
Kartu utama + stepper, riwayat jadwal, kartu per aset dengan tombol Normal/Rusak dan keterangan, pernyataan, tombol **Terima & Kirim Checklist**.

### 26.3 Admin — Daftar (`HalamanAssetAdminOnboardingRuangan`)
Tab Proses/Selesai, kartu transaksi, tombol Tambah Alokasi.

### 26.4 Admin — Form (`HalamanAssetAdminAddOnboardingRuangan`)
Seragam dengan form onboarding karyawan: Ruangan Tujuan, PIC Ruangan (terkunci bila ada PIC master), Asset On Boarding (pemilih kategori → asset), Jadwal, Catatan, **Kirim ke PIC Ruangan**.

### 26.5 Admin — Detail (`HalamanAssetAdminDetailOnboardingRuangan`)
Seragam dengan detail klaim garansi: kartu utama bergradien, stepper (Dijadwalkan, Serah Terima, Review, Selesai), timeline vertikal per aset (`AssetRuanganAsetTimeline`: Penjadwalan, Serah Terima & Checklist, Review, Selesai), kartu pengganti, informasi ruangan, riwayat jadwal. Tombol bawah: **Ubah Jadwal** / **Jadwalkan Pengganti** / **Konfirmasi Selesai** sesuai fase.

### 26.6 Entry point
Kartu **PIC Ruangan** di `HalamanHomeAsset`; tombol **On Boarding** di `HalamanAssetAdminRuangan`; notifikasi (`asset_pic_ruangan_onboarding`, `asset_admin_detail_onboarding_ruangan` terdaftar di `assetNotifRoutes`).

## 27. Mobile → API

| Layar | Aksi | API |
|---|---|---|
| PIC home | muat | API-008 |
| PIC detail | muat / kirim | API-009 / API-010 |
| Admin daftar | muat per tab | API-002 |
| Admin form | opsi / kirim | API-001 / API-003 |
| Admin detail | muat | API-004 |
| Admin detail | ubah jadwal / pengganti / konfirmasi | API-005 / API-006 / API-007 |

## 28. API → Mobile

Kunci JSON dipetakan ke model (`AssetPicRuanganDetail`, `AssetAdminRuanganDetail`, dll.). `fase` menentukan stepper dan tombol; flag `bisa_*` menentukan aksi; `perlu_pengganti[]` membentuk kartu pengganti.

## 29. Mapping Mobile → API → Database

| Mobile | API | Database |
|---|---|---|
| Field Ruangan Tujuan | `id_asset_lokasi_m` | `asset_trx_onboarding_ruangan.id_asset_lokasi_m` |
| Field PIC (hanya ruangan tanpa PIC) | `id_karyawan_pic` | `asset_lokasi_pic_m` (baru) + `id_karyawan_pic` |
| Tambah Asset | `asset_ids[]` | `_item` (satu baris per aset, ronde 1) |
| Jadwal | `tanggal`, `jam` | `tgl/jam_serah_terima` + `_jadwal` |
| Normal/Rusak + keterangan | `items[]` | `_item.status_item`, `ket_rusak` |
| Pilih pengganti | `items[].id_asset_data` | `_item` ronde+1 (`parent_id`) |

## 30. Mapping Database → API → Mobile

| Database | API | Mobile |
|---|---|---|
| `_jadwal.tgl_terima_pic` | `fase`, `jadwal[].tgl_terima_pic` | Stepper, "Diterima …" |
| `_item.parent_id` | `diganti`, `parent_id` | Badge "Diganti", "Digantikan oleh/Pengganti untuk" |
| `asset_lokasi_pic_m` | `ruangan[].id_karyawan_pic` | PIC terisi & terkunci |
| `asset_data` (pemegang ruangan) | `ruangan[].assets[]` | Daftar aset di halaman PIC |

## 31. Validation

### 31.1 Mobile
Form: ruangan, PIC, minimal satu asset, tanggal, jam wajib (error muncul setelah submit). Detail PIC: tombol kirim aktif bila semua aset dinilai dan keterangan rusak terisi. Detail admin: tombol pengganti aktif bila semua pengganti dan jadwal terisi.

### 31.2 Backend
Aturan Laravel: `asset_ids` array min 1 distinct; `tanggal` `date_format:Y-m-d`; `jam` `date_format:H:i`; `items.*.status_item` `in:normal,rusak`; `ket_rusak` wajib bila rusak. Aturan bisnis: BR-001 s.d. BR-010.

### 31.3 Database
Unique PIC per ruangan; PK; FK jadwal dibuat bila tipe/charset tabel cocok.

## 32. Error Handling

| Kode | Kondisi |
|---|---|
| `UNAUTHORIZED` | Tanpa konteks login |
| `FORBIDDEN` | Bukan admin asset |
| `NOT_FOUND` | Transaksi tidak ada / bukan milik PIC |
| `VALIDATION_ERROR` | Payload tidak valid |
| `BUSINESS_RULE_VIOLATION` (422) | Pelanggaran aturan bisnis (pesan Indonesia di §9) |
| `SYSTEM_ERROR` | Kesalahan tak terduga (dicatat ke log) |

Controller wajib mengimpor `InvalidArgumentException` dan `Throwable`.

## 33. Audit Trail

`user_input`/`user_update` = `nik - nama`; `tgl_approval` + `user_approval` saat konfirmasi; riwayat jadwal append-only; `asset_trx_log` (`onboarding_ruangan`) per aset saat konfirmasi.

## 34. Notification

| Event | Penerima | Rute tujuan |
|---|---|---|
| `onboarding_ruangan.dikirim` | PIC Ruangan | `asset_pic_ruangan_onboarding` |
| `onboarding_ruangan.jadwal_diubah` | PIC Ruangan | idem |
| `onboarding_ruangan.pengganti` | PIC Ruangan | idem |
| `onboarding_ruangan.selesai` | PIC Ruangan | idem |
| `onboarding_ruangan.diterima` | Seluruh admin asset | `asset_admin_detail_onboarding_ruangan` |

Penerima PIC diambil dari `id_karyawan_pic` header (penerima baru `PIC_RUANGAN`). Dikirim setelah transaksi database selesai.

## 35. Security

JWT; admin dicek lewat `asset_admin`; PIC dibatasi per transaksi; semua query memakai `id_company`; kunci per company mencegah balapan penomoran/alokasi.

## 36. Manual Book User/Admin

### A. Admin/HR — Membuat alokasi
1. Menu Asset → Ruangan → **On Boarding** → **Tambah Alokasi**.
2. Pilih **Ruangan Tujuan**. PIC terisi otomatis dan terkunci; bila ruangan belum punya PIC, pilih karyawan (akan tersimpan sebagai PIC ruangan).
3. **Tambah Asset**: pilih kategori lalu asset (ulangi untuk asset lain).
4. Pilih tanggal dan jam serah terima, isi catatan bila perlu, tekan **Kirim ke PIC Ruangan**.

### B. PIC Ruangan — Menerima aset
1. Buka menu **PIC Ruangan** di halaman pertama Asset; ketuk banner **Serah Terima Asset Ruangan**.
2. Setelah aset diserahkan, periksa tiap aset: pilih **Normal** atau **Rusak** (isi keterangan bila rusak).
3. Tekan **Terima & Kirim Checklist**. Halaman menjadi hanya-baca sampai Admin/HR mereview.

### C. Admin/HR — Review
1. Buka transaksi di tab **Proses**; periksa timeline tiap aset.
2. Semua normal: tekan **Konfirmasi Selesai**.
3. Ada rusak: pada kartu pengganti pilih asset pengganti untuk **setiap** aset rusak, tetapkan jadwal baru, tekan **Jadwalkan Pengganti**. PIC mengecek unit pengganti; ulangi review.

### D. Admin/HR — Mengubah jadwal
Selama PIC belum menerima, tekan **Ubah Jadwal**, isi tanggal, jam, dan alasan (opsional).

## 37. Kondisi Button

| Tombol | Aktif bila |
|---|---|
| Kirim ke PIC Ruangan | Form tidak sedang mengirim (validasi saat ditekan) |
| Terima & Kirim Checklist | Fase `menunggu_serah_terima`, semua aset dinilai, keterangan rusak terisi |
| Ubah Jadwal | `bisa_ubah_jadwal` (PIC belum menerima) |
| Jadwalkan Pengganti | Ada aset rusak, semua pengganti + tanggal + jam terisi |
| Konfirmasi Selesai | `bisa_konfirmasi` (menunggu review, tanpa rusak) |

## 38. Testing Scenario

| ID | Skenario | Hasil |
|---|---|---|
| T-01 | Opsi form | Aset tersedia, kategori, ruangan aktif belum onboarding |
| T-02 | Buat valid | Aset `proses`, item per aset, jadwal ronde 1 |
| T-03 | Ruangan tidak aktif/tanpa PIC/jadwal lampau | Ditolak |
| T-04 | PIC master dikunci; ruangan tanpa PIC menyimpan ke master | Sesuai |
| T-05 | Ruangan sudah onboarding (proses/selesai) | Ditolak dan hilang dari opsi |
| T-06 | Ubah jadwal sebelum/sesudah PIC terima | Diizinkan / ditolak |
| T-07 | PIC terima: kurang item, rusak tanpa keterangan, PIC lain | Ditolak |
| T-08 | Rusak → pengganti → ronde 2 → konfirmasi | Aset rusak `rusak`, pengganti `terpakai` |
| T-09 | Konfirmasi tanpa checklist / dengan rusak | Ditolak |
| T-10 | Mobile: form, daftar tab, detail aksi, halaman PIC | Lulus (18 test mobile) |

## 39. Temuan Ketidaksesuaian

| ID | Temuan |
|---|---|
| INC-01 | Dokumen bisnis: checklist per **Daftar Atribut Aset** (Aset Utama/Kelengkapan, "Atribut Lainnya"). Implementasi: checklist **per aset utama** (keputusan pemilik produk), seperti KRY01 |
| INC-02 | Dokumen bisnis: Admin/HR dapat mengganti PIC. Implementasi: PIC dari master **terkunci**; hanya ruangan tanpa PIC yang dipilihkan karyawan dan pilihannya disimpan ke master |
| INC-03 | Dokumen bisnis menamai status aset selama alokasi `DIALOKASIKAN`. Implementasi memakai `proses` (nilai enum status aset tidak ditambah; keputusan pemilik produk) |
| INC-04 | Satu ruangan hanya boleh onboarding satu kali (aturan tambahan di luar dokumen); penambahan aset ke ruangan aktif adalah alur terpisah ALK-RGN03 |
| INC-05 | Dokumen bisnis menyebut "menarik item rusak ke gudang" tanpa lokasi; implementasi menandai aset `rusak` dan tidak mengubah `id_asset_lokasi_m` (NULL) |
| INC-06 | Enum `status_onboarding` memuat `draft`/`ditolak` (dari KRY01) yang tidak dipakai alur ini |

## 40. Edge Case

- Dua admin membuat alokasi untuk aset yang sama: pengunci company + validasi `tersedia`, yang kedua ditolak.
- Aset berubah status di luar alur saat konfirmasi: ditolak "Satu atau beberapa asset tidak dapat diserahkan ke ruangan."
- Admin memilih pengganti yang sudah dipakai alur lain: ditolak "Satu atau beberapa asset pengganti tidak tersedia."
- Pengganti ikut rusak: ronde berikutnya (loop tanpa batas sampai normal atau stok habis).
- Ruangan tanpa PIC: admin wajib memilih karyawan; setelah tersimpan, onboarding ruangan itu tidak bisa dibuat lagi (BR-003).

## 41. Potential Bug / Technical Risk

### RISK-001 — Skema dari SQL manual (TINGGI)
Wajib menjalankan blok 5 dan 5b `sql-alk-rgn01-onboarding-ruangan.sql` pada setiap environment; tanpa itu kolom lama (`jenis_item` dst.) membuat insert gagal.

### RISK-002 — Tidak ada pembatalan alokasi (SEDANG)
Alokasi yang tertahan mengunci aset (`proses`) dan ruangannya (BR-003). Belum ada aksi batal/hapus.

### RISK-003 — Notifikasi di luar transaksi (RENDAH)
Kegagalan push tidak membatalkan transaksi (dicatat ke log).

### RISK-004 — Kapasitas penomoran 999/bulan (RENDAH)

### RISK-005 — PIC tersimpan permanen (SEDANG)
PIC yang dipilih manual menjadi master. Mengganti PIC ruangan hanya lewat data master (belum ada layar).

### RISK-006 — Belum diuji di perangkat (SEDANG)
Test memakai database tiruan dan widget test; belum diverifikasi di emulator/server nyata, termasuk push notifikasi.

## 42. Dependency

`Asset_m/Asset_c`, `AssetNotification`, `asset_admin`, `asset_lokasi_m`, `asset_lokasi_pic_m`, `asset_kategori_m`, `asset_data`, `asset_trx_log`, `data_karyawan`, `master_cabang`, `master_departemen`; mobile: pemilih aset/karyawan dari `HalamanAssetAdminAddOnboardingKaryawan`, `AssetPemeliharaanStepper`, `AssetPeminjamanCard`.

## 43. Source Code Reference

| Area | Lokasi |
|---|---|
| Model | `API/app/Models/Asset/AssetAdminRuangan_m.php`, `AssetLokasiPic_m.php` |
| Controller | `API/app/Http/Controllers/Asset/Admin/AssetAdminOnboardingRuangan_c.php`, `Karyawan/AssetKaryawanPicRuangan_c.php` |
| Notifikasi | `API/app/Support/Asset/AssetNotification.php` |
| SQL | `Claude outputs/sql-alk-rgn01-onboarding-ruangan.sql` |
| Test API | `API/tests/Asset/AssetAdminRuanganTest.php`, `AssetLokasiPicTest.php` |
| Mobile UI | `MOBILE/lib/UI/Modul/Asset/PicRuangan/`, `AssetAdmin/AssetAdminRuangan/` |
| Test mobile | `MOBILE/test/asset/asset_admin_ruangan_test.dart`, `asset_pic_ruangan_test.dart` |

## 44. Developer Notes

- Fase tidak disimpan: turunkan dari `status_approval` + `tgl_terima_pic` jadwal ronde terakhir (`AssetLokasiPic_m::fase`).
- `parent_id` item = rantai penggantian seperti KRY01; jangan dipakai untuk pengelompokan.
- Test model memakai SQLite in-memory dengan skema tiruan; tambahkan kolom ke skema test bila model membaca kolom baru.
- Test layar tinggi/lebar: dialog time picker pada font uji lebih lebar dari 360px, gunakan surface 800px.
- `tests/Routing` (4 test jumlah route) sudah gagal sebelum fitur ini; bukan akibat ALK-RGN01.

## 45. End-to-End Traceability

| Kebutuhan | Rule | API | Mobile | DB |
|---|---|---|---|---|
| Hanya aset tersedia | BR-001 | API-001/003 | Pemilih aset | `asset_data.status_asset` |
| PIC otomatis/terkunci | BR-004 | API-001/003 | Field PIC | `asset_lokasi_pic_m` |
| Sekali onboarding | BR-003 | API-001/003 | Pilihan ruangan | `asset_trx_onboarding_ruangan` |
| Checklist PIC | BR-007 | API-010 | Detail PIC | `_item.status_item` |
| Pengganti | BR-008 | API-006 | Kartu pengganti | `_item.parent_id`, `ronde` |
| Konfirmasi | BR-009 | API-007 | Tombol konfirmasi | `asset_data`, `asset_trx_log` |

### 45.4 Walkthrough Lengkap dengan Contoh Data

Walkthrough berikut mengikuti satu alokasi aset ke ruangan dari awal sampai selesai, termasuk satu ronde penggantian karena ada aset yang rusak. Setiap langkah menjelaskan apa yang dilakukan di mobile, apa yang dikirim, apa yang diproses API, apa yang tersimpan di database, dan apa yang diterima mobile. Data bersifat **sintetis**, tetapi nama kolom, urutan proses, dan pesan diambil dari kode (`AssetAdminRuangan_m`, `AssetLokasiPic_m`, `AssetAdminOnboardingRuangan_c`, `AssetKaryawanPicRuangan_c`, `AssetNotification`).

| Langkah | Pelaku | Aksi | Endpoint |
|:---:|---|---|---|
| 0 | Admin/HR | Membuka form **Tambah Alokasi**, memuat opsi | API-001 `GET /asset/onboarding-ruangan/options` |
| 1 | Admin/HR | **Kirim ke PIC Ruangan** (2 aset, ruangan tanpa PIC) | API-003 `POST /asset/onboarding-ruangan` |
| 2 | Admin/HR | **Ubah Jadwal** sebelum PIC menerima | API-005 `PUT /asset/onboarding-ruangan/{id}/jadwal` |
| 3 | PIC Ruangan | Memeriksa fisik, mengirim checklist (1 normal, 1 rusak) | API-009 lalu API-010 `POST /asset/karyawan/pic-ruangan/onboarding/{id}/terima` |
| 4 | Admin/HR | **Jadwalkan Pengganti** untuk aset rusak | API-006 `POST /asset/onboarding-ruangan/{id}/pengganti` |
| 5 | PIC Ruangan | Mengirim checklist ronde 2 (normal) | API-010 |
| 6 | Admin/HR | **Konfirmasi Selesai** | API-007 `POST /asset/onboarding-ruangan/{id}/konfirmasi` |

#### Data awal

Company `CMP001`. Tanggal 22 Oktober 2026, periode ID `202610`.

**`data_karyawan`** (semua aktif)

| id_karyawan | nik | nama_lengkap | Peran |
|---|---|---|---|
| KRY0099 | ADM-001 | Siti Admin | Admin Asset (aktor Admin/HR) |
| KRY0030 | EMP-030 | Dewi PIC | Akan menjadi PIC Ruangan |

**`asset_lokasi_m`**: `LOK0007` "Ruang Meeting Lt.2", `tipe = ruangan`, `status_aktif = 1`, cabang "Jakarta", departemen "Operasional". Ruangan ini **belum punya baris** di `asset_lokasi_pic_m` dan belum pernah di-onboarding.

**`asset_data`** (semua `status_asset = tersedia`, `pemegang_asset = gudang`, `id_asset_lokasi_m = LOK0001` gudang)

| id_asset_data | nama_asset | Peran dalam skenario |
|---|---|---|
| AST0101 | Laptop Dell 14 | Aset awal, kondisi normal |
| AST0102 | Monitor LG 24" | Aset awal, ternyata rusak |
| AST0103 | Monitor Samsung 24" | Pengganti monitor |

Penomoran: header, item, jadwal, dan master PIC memakai `id_company + YYYYMM + 3 digit` (urut per tabel); `asset_trx_log` dan notifikasi memakai 5 digit. `no_trx = RGN01-YYYYMMNNN`.

---

#### Langkah 0 — Admin membuka form Tambah Alokasi

**Mobile:** Dashboard Ruangan → tombol **On Boarding** → **Tambah Alokasi** (`HalamanAssetAdminAddOnboardingRuangan`). `AssetAdminRuanganFormBloc` memuat opsi lewat `assetAdminRuanganApi`.

**Dikirim**
```http
GET /asset/onboarding-ruangan/options
Authorization: Bearer <access_token_admin>
```

**Diproses API:** `getOptions` hanya membaca: aset `tersedia` di gudang beserta pohon kategori, ruangan aktif yang **belum** punya transaksi `proses`/`dikonfirmasi` beserta PIC masternya (bila ada), dan karyawan aktif. **Database:** hanya `SELECT`.

**Diterima mobile:** `LOK0007` muncul dengan `id_karyawan_pic` kosong, sehingga field **PIC Ruangan** terbuka untuk dipilih; bila ruangan sudah punya PIC master, field terkunci. Ketiga aset muncul di pemilih kategori → asset.

#### Langkah 1 — Admin mengalokasikan 2 aset ke Ruang Meeting Lt.2

**Mobile:** Admin memilih ruangan `LOK0007`, memilih Dewi sebagai PIC, mencentang Laptop dan Monitor LG, mengisi jadwal 23 Okt 2026 pukul 09:00 dan catatan, lalu menekan **Kirim ke PIC Ruangan**.

**Dikirim**
```http
POST /asset/onboarding-ruangan
Authorization: Bearer <access_token_admin>
```
```json
{ "id_asset_lokasi_m": "LOK0007",
  "asset_ids": ["AST0101", "AST0102"],
  "id_karyawan_pic": "KRY0030",
  "tanggal": "2026-10-23", "jam": "09:00",
  "catatan": "Bawa kunci ruangan" }
```

**Diproses API** (`storeOnboardingRuangan` → `AssetAdminRuangan_m::store`, satu `DB::transaction(…, 3)`)
1. Validasi controller: `id_asset_lokasi_m` wajib ("Ruangan tujuan wajib dipilih."), `asset_ids` minimal satu dan tidak boleh kembar ("Minimal satu asset wajib dipilih." / "Asset yang sama tidak boleh dipilih lebih dari sekali."), `tanggal` `Y-m-d`, `jam` `H:i`.
2. `lockCompany` (`SELECT … FOR UPDATE` pada `master_company`), lalu cek aktor terdaftar sebagai admin asset ("Pengguna tidak terdaftar sebagai admin asset.").
3. `findRuanganAktif`: `tipe = ruangan` dan `status_aktif = 1` ("Ruangan tujuan tidak ditemukan atau tidak aktif.").
4. Cek sekali-onboarding: tidak boleh ada transaksi `proses`/`dikonfirmasi` untuk ruangan itu ("Ruangan ini sudah pernah di-onboarding. Onboarding hanya dapat dilakukan satu kali per ruangan.").
5. `resolvePic`: PIC master ada → dipakai dan tidak boleh diganti ("PIC Ruangan sudah ditetapkan pada ruangan ini dan tidak dapat diganti."); tidak ada master dan tidak ada pilihan → "PIC Ruangan belum ditetapkan untuk ruangan ini."; karyawan tidak aktif → "PIC Ruangan tidak ditemukan atau tidak aktif.". Di sini master kosong dan Dewi dipilih, jadi PIC dipakai dan **akan disimpan ke master**.
6. `getLockedAssets` mengunci aset (`FOR UPDATE`) yang `tersedia` dan berada di gudang; jumlahnya harus sama dengan yang diminta ("Satu atau beberapa asset tidak tersedia untuk dialokasikan.").
7. `parseJadwalValue`: tanggal dan jam wajib, format benar, tidak sebelum hari ini ("Tanggal serah terima tidak boleh sebelum tanggal pengajuan.").
8. Nomor: `nextSequence` + `ensureSequenceCapacity` → `CMP001202610001`, `RGN01-202610001`.
9. PIC manual → INSERT `asset_lokasi_pic_m`; INSERT header, 2 item ronde 1, 1 jadwal ronde 1; UPDATE `asset_data` → `proses`.
10. Setelah commit, controller memanggil `AssetNotification::send('onboarding_ruangan.dikirim', …)` ke PIC.

**Hasil database**

`asset_lokasi_pic_m` (baris baru, karena ruangan belum punya PIC)

| Kolom | Nilai |
|---|---|
| id_asset_lokasi_pic_m | CMP001202610001 |
| id_asset_lokasi_m / id_karyawan | LOK0007 / KRY0030 |
| id_company / tgl_input / user_input | CMP001 / 2026-10-22 09:00:00 / ADM-001 - Siti Admin |

`asset_trx_onboarding_ruangan` (1 baris)

| Kolom | Nilai |
|---|---|
| id_asset_trx_onboarding_ruangan / no_trx | CMP001202610001 / RGN01-202610001 |
| tgl_onboarding | 2026-10-22 |
| id_asset_lokasi_m / nama_asset_ruangan | LOK0007 / Ruang Meeting Lt.2 |
| nama_cabang / nama_departemen | Jakarta / Operasional |
| id_asset_lokasi_pic_m | CMP001202610001 |
| id_karyawan_pic / nama_pic / nik_pic | KRY0030 / Dewi PIC / EMP-030 |
| tgl_serah_terima / jam_serah_terima | 2026-10-23 / 09:00:00 |
| keterangan_serah_terima | Bawa kunci ruangan |
| id_karyawan / nik_karyawan / nama_karyawan | KRY0099 / ADM-001 / Siti Admin |
| status_approval | **proses** |
| user_approval / tgl_approval | `''` / NULL |
| tgl_input / user_input | 2026-10-22 09:00:00 / ADM-001 - Siti Admin |

`asset_trx_onboarding_ruangan_item` (2 baris, ronde 1)

| id_…_item | id_asset_data | nama_asset | ronde | parent_id | status_item | ket_rusak |
|---|---|---|:---:|---|---|---|
| CMP001202610001 | AST0101 | Laptop Dell 14 | 1 | `''` | belum_dicek | `''` |
| CMP001202610002 | AST0102 | Monitor LG 24" | 1 | `''` | belum_dicek | `''` |

`asset_trx_onboarding_ruangan_jadwal` (1 baris)

| id_…_jadwal | ronde | tanggal | jam | tgl_terima_pic | keterangan |
|---|:---:|---|---|---|---|
| CMP001202610001 | 1 | 2026-10-23 | 09:00:00 | NULL | `''` |

`asset_data` (AST0101 dan AST0102)

| Kolom | Sebelum | Sesudah |
|---|---|---|
| status_asset | tersedia | **proses** |
| pemegang_asset / id_asset_lokasi_m | gudang / LOK0001 | gudang / LOK0001 (tidak berubah) |
| tgl_update / user_update | NULL / `''` | 2026-10-22 09:00:00 / ADM-001 - Siti Admin |

`notif_data` (1 baris untuk Dewi)

| Kolom | Nilai |
|---|---|
| id_karyawan / jenis / id_ref | KRY0030 / asset / CMP001202610001 |
| rute | asset_pic_ruangan_onboarding |
| ket_notif_id | Anda ditunjuk menerima asset RGN01-202610001 untuk Ruang Meeting Lt.2. Cek jadwal serah terima. |
| ket_notif_en | You were assigned to receive assets RGN01-202610001 for Ruang Meeting Lt.2. Check the handover schedule. |
| flag_baca | 0 |

Push FCM ikut dikirim ke perangkat Dewi; kegagalan push hanya dicatat di log dan tidak membatalkan transaksi.

**Diterima mobile** (HTTP 201)
```json
{ "code": "CREATED",
  "message": "Alokasi asset ke ruangan berhasil dikirim ke PIC Ruangan.",
  "data": { "id": "CMP001202610001", "no_trx": "RGN01-202610001", "status_approval": "proses" } }
```
Form menutup, daftar tab **Proses** dimuat ulang. Fase turunan: `menunggu_serah_terima` (jadwal ronde 1 belum punya `tgl_terima_pic`).

#### Langkah 2 — Admin mengubah jadwal

Ruangan belum siap, jadwal digeser ke 24 Okt 2026 pukul 13:30.

**Mobile:** detail transaksi → **Ubah Jadwal** (tombol hanya muncul bila `bisa_ubah_jadwal = true`).

**Dikirim**
```http
PUT /asset/onboarding-ruangan/CMP001202610001/jadwal
```
```json
{ "tanggal": "2026-10-24", "jam": "13:30", "keterangan": "Ruangan belum siap" }
```

**Diproses API** (`updateJadwalOnboardingRuangan` → `updateJadwal`)
1. Validasi `tanggal`, `jam` ("Tanggal serah terima wajib diisi." / "Format tanggal serah terima tidak valid." dan padanan jam), `keterangan` maksimal 255.
2. `lockCompany`, admin, `lockHeader` (header harus ada dan `status_approval = proses`: "Onboarding ruangan tidak ditemukan." / "Onboarding ruangan sudah selesai.").
3. `rondeSaatIni` = `MAX(ronde)` item = 1; fase harus `menunggu_serah_terima` ("Jadwal hanya dapat diubah sebelum PIC Ruangan menerima asset.").
4. `parseJadwalValue`, INSERT jadwal baru (**append-only**, ronde yang sama), UPDATE jadwal terkini di header.
5. Controller mengirim `onboarding_ruangan.jadwal_diubah` ke PIC.

**Hasil database**

`asset_trx_onboarding_ruangan_jadwal` (baris baru; baris lama tetap sebagai riwayat)

| id_…_jadwal | ronde | tanggal | jam | tgl_terima_pic | keterangan |
|---|:---:|---|---|---|---|
| CMP001202610001 | 1 | 2026-10-23 | 09:00:00 | NULL | `''` |
| **CMP001202610002** | 1 | **2026-10-24** | **13:30:00** | NULL | Ruangan belum siap |

`asset_trx_onboarding_ruangan` (header): `tgl_serah_terima = 2026-10-24`, `jam_serah_terima = 13:30:00`, `tgl_update = 2026-10-22 10:15:00`, `user_update = ADM-001 - Siti Admin`. Item, `asset_data`, dan status tidak berubah.

`notif_data`: satu baris baru untuk KRY0030, judul "Jadwal Serah Terima Diperbarui", isi "Jadwal serah terima RGN01-202610001 telah diperbarui.".

**Diterima mobile** (HTTP 200): `{ "code": "UPDATED", "message": "Jadwal serah terima berhasil diperbarui.", "data": { "id": "CMP001202610001", "no_trx": "RGN01-202610001" } }`. Detail menampilkan dua entri **Riwayat Jadwal**; jadwal aktif adalah entri terakhir.

#### Langkah 3 — PIC memeriksa fisik dan mengirim checklist

Pada 24 Okt, Dewi menerima dua aset. Laptop berfungsi normal, monitor layarnya bergaris.

**Mobile:** notifikasi membuka `HalamanDetailOnboardingPicRuangan`. `AssetPicRuanganDetailBloc` memuat detail (API-009). Dewi menekan **Normal** pada Laptop, **Rusak** pada Monitor dengan keterangan, mencentang pernyataan, lalu **Terima & Kirim Checklist**.

**Dikirim**
```http
GET  /asset/karyawan/pic-ruangan/onboarding/CMP001202610001
POST /asset/karyawan/pic-ruangan/onboarding/CMP001202610001/terima
Authorization: Bearer <access_token_dewi>
```
```json
{ "items": [
  { "id_item": "CMP001202610001", "status_item": "normal" },
  { "id_item": "CMP001202610002", "status_item": "rusak", "ket_rusak": "Layar bergaris vertikal" } ] }
```

**Diproses API** (`terimaOnboardingRuangan` → `AssetLokasiPic_m::terimaOnboarding`)
1. Validasi controller: `items` wajib dan tiap `id_item` tidak kembar; `status_item` hanya `normal` atau `rusak`; `ket_rusak` wajib bila `rusak`.
2. `lockCompany`; header dicari dengan filter `id_karyawan_pic` = pengguna login dan dikunci `FOR UPDATE`. Transaksi milik PIC lain → "Onboarding ruangan tidak ditemukan.".
3. Fase harus `menunggu_serah_terima` ("Onboarding ruangan sudah tidak menunggu serah terima."); karyawan harus aktif.
4. `itemChecklist` mengambil item ronde berjalan (ronde 1: 2 item). Semua item **wajib** dicek, tidak kurang dan tidak berlebih ("Seluruh item pada serah terima wajib dicek."). Rusak tanpa keterangan → "Keterangan kerusakan wajib diisi.".
5. UPDATE `status_item` dan `ket_rusak` per item (`ket_rusak` dikosongkan bila normal), isi `tgl_terima_pic` pada jadwal ronde terakhir, sentuh `tgl_update` header.
6. Controller mengirim `onboarding_ruangan.diterima` ke semua admin asset.

**PIC tidak mengubah `asset_data`.** Aset tetap `proses`; keputusan ada di Admin/HR.

**Hasil database**

`asset_trx_onboarding_ruangan_item`

| id_…_item | status_item | ket_rusak |
|---|---|---|
| CMP001202610001 (Laptop) | belum_dicek → **normal** | `''` |
| CMP001202610002 (Monitor LG) | belum_dicek → **rusak** | Layar bergaris vertikal |

`asset_trx_onboarding_ruangan_jadwal`: baris `CMP001202610002` kini `tgl_terima_pic = 2026-10-24 13:40:00`. Fase turunan menjadi **menunggu_review**.

`asset_trx_onboarding_ruangan` (header): hanya `tgl_update = 2026-10-24 13:40:00`, `user_update = EMP-030 - Dewi PIC`.

`notif_data`: satu baris per admin asset (misalnya KRY0099), rute `asset_admin_detail_onboarding_ruangan`, judul "Serah Terima Diterima PIC", isi "Dewi PIC telah menerima asset RGN01-202610001 dan mengirim checklist kondisi.".

**Diterima mobile** (HTTP 200)
```json
{ "code": "UPDATED",
  "message": "Serah terima dikirim. Kerusakan akan ditindaklanjuti admin.",
  "data": { "id": "CMP001202610001", "no_trx": "RGN01-202610001",
            "ada_rusak": true, "fase": "menunggu_review" } }
```
Bila tidak ada yang rusak, pesannya "Serah terima berhasil dikirim." dan `ada_rusak = false`. Layar PIC beralih ke status menunggu review.

#### Langkah 4 — Admin menjadwalkan pengganti

**Mobile:** detail transaksi menampilkan timeline per aset dan `perlu_pengganti[]` berisi Monitor LG. Admin menekan **Jadwalkan Pengganti**, memilih Monitor Samsung sebagai pengganti, mengisi jadwal 26 Okt pukul 10:00.

**Dikirim**
```http
POST /asset/onboarding-ruangan/CMP001202610001/pengganti
```
```json
{ "items": [ { "id_item": "CMP001202610002", "id_asset_data": "AST0103" } ],
  "tanggal": "2026-10-26", "jam": "10:00", "keterangan": "Unit pengganti monitor" }
```

**Diproses API** (`jadwalkanPenggantiOnboardingRuangan` → `jadwalkanPengganti`)
1. Validasi: `items` wajib ("Asset pengganti wajib dipilih."), `id_asset_data` tidak kembar ("Asset pengganti yang sama tidak boleh dipilih lebih dari sekali.").
2. `lockCompany`, admin, `lockHeader`, `rondeSaatIni` = 1. Fase harus `menunggu_review` ("Pengganti hanya dapat dijadwalkan setelah PIC Ruangan mengirim checklist.").
3. `asetRusakRondeIni`: item ronde ini berstatus `rusak` yang belum digantikan. Kosong → "Tidak ada asset rusak yang perlu diganti.".
4. Pilihan admin harus cocok **penuh** dengan daftar rusak, tidak kurang dan tidak lebih ("Seluruh asset rusak wajib diberi asset pengganti."); pengganti tidak boleh kembar ("Asset pengganti yang sama tidak boleh dipilih lebih dari sekali.").
5. `getLockedAssets` untuk pengganti: harus `tersedia` di gudang ("Satu atau beberapa asset pengganti tidak tersedia.").
6. `parseJadwalValue`.
7. UPDATE aset rusak → `rusak`; INSERT item ronde 2 dengan `parent_id` = id item rusak; INSERT jadwal ronde 2; UPDATE header; UPDATE pengganti → `proses`.
8. Controller mengirim `onboarding_ruangan.pengganti` ke PIC.

**Hasil database**

`asset_data`

| id_asset_data | Kolom | Sebelum | Sesudah |
|---|---|---|---|
| AST0102 (Monitor LG) | status_asset | proses | **rusak** |
| | ket_rusak | `''` | Layar bergaris vertikal |
| | pemegang_asset / id_pemegang_asset | gudang / `''` | gudang / `''` (tetap di gudang) |
| AST0103 (Monitor Samsung) | status_asset | tersedia | **proses** |

`asset_trx_onboarding_ruangan_item` (baris baru, ronde 2)

| id_…_item | id_asset_data | ronde | parent_id | status_item |
|---|---|:---:|---|---|
| CMP001202610003 | AST0103 | **2** | CMP001202610002 | belum_dicek |

Baris `CMP001202610002` tetap ada sebagai bukti riwayat; karena sudah menjadi `parent_id` baris lain, ia dihitung **sudah digantikan** (`diganti = true`).

`asset_trx_onboarding_ruangan_jadwal`

| id_…_jadwal | ronde | tanggal | jam | tgl_terima_pic | keterangan |
|---|:---:|---|---|---|---|
| CMP001202610003 | **2** | 2026-10-26 | 10:00:00 | NULL | Unit pengganti monitor |

Header: `tgl_serah_terima = 2026-10-26`, `jam_serah_terima = 10:00:00`. Fase turunan kembali ke **menunggu_serah_terima** (jadwal ronde terakhir belum diterima). Status header tetap `proses`.

`notif_data`: baris untuk KRY0030, judul "Asset Pengganti", isi "Asset pengganti untuk RGN01-202610001 dijadwalkan. Cek jadwal serah terima baru.".

**Diterima mobile** (HTTP 200)
```json
{ "code": "UPDATED", "message": "Asset pengganti berhasil dijadwalkan ke PIC Ruangan.",
  "data": { "id": "CMP001202610001", "no_trx": "RGN01-202610001", "ronde": 2, "jumlah_pengganti": 1 } }
```

#### Langkah 5 — PIC mengirim checklist ronde 2

Pada 26 Okt, Dewi menerima monitor pengganti. `itemChecklist` untuk ronde 2 hanya berisi **satu** item (Monitor Samsung); Laptop yang sudah dinilai di ronde 1 tidak perlu dicek ulang.

**Dikirim**
```http
POST /asset/karyawan/pic-ruangan/onboarding/CMP001202610001/terima
```
```json
{ "items": [ { "id_item": "CMP001202610003", "status_item": "normal" } ] }
```

**Diproses API:** sama seperti langkah 3, tetapi dengan `ronde = 2`. Mengirim item ronde 1 di sini tidak diterima karena jumlah dan identitas item harus sama persis dengan checklist ronde berjalan ("Seluruh item pada serah terima wajib dicek.").

**Hasil database:** item `CMP001202610003` → `normal`; jadwal `CMP001202610003` → `tgl_terima_pic = 2026-10-26 10:20:00`; header `tgl_update`/`user_update` diperbarui. Fase turunan **menunggu_review**. Admin dinotifikasi (`onboarding_ruangan.diterima`).

**Diterima mobile** (HTTP 200): `message` "Serah terima berhasil dikirim.", `data.ada_rusak = false`, `data.fase = "menunggu_review"`.

#### Langkah 6 — Admin mengonfirmasi selesai

**Mobile:** detail transaksi menampilkan `bisa_konfirmasi = true` (fase menunggu review dan tidak ada aset rusak pada ronde ini). Admin menekan **Konfirmasi Selesai**.

**Dikirim**
```http
POST /asset/onboarding-ruangan/CMP001202610001/konfirmasi
```
(tanpa body)

**Diproses API** (`konfirmasiOnboardingRuangan` → `konfirmasi`)
1. `lockCompany`, admin, `lockHeader`, `rondeSaatIni` = 2.
2. Fase harus `menunggu_review` ("Onboarding ruangan belum menunggu review Admin/HR.").
3. Tidak boleh ada aset rusak pada ronde berjalan ("Masih ada asset rusak. Jadwalkan penggantian sebelum menyelesaikan alokasi.").
4. `asetAktif`: semua baris item yang **belum digantikan** → Laptop (AST0101) dan Monitor Samsung (AST0103). Monitor LG (AST0102) tidak ikut.
5. Aset aktif dikunci `FOR UPDATE`; semuanya harus `proses` dan `gudang` ("Satu atau beberapa asset tidak dapat diserahkan ke ruangan.").
6. UPDATE `asset_data` → `terpakai`, `pemegang_asset = ruangan`, `id_pemegang_asset` dan `id_asset_lokasi_m` = `LOK0007`.
7. INSERT satu `asset_trx_log` per aset (`insertAssetLog`).
8. UPDATE header → `dikonfirmasi`.
9. Controller mengirim `onboarding_ruangan.selesai` ke PIC.

**Hasil database**

`asset_data`

| id_asset_data | Kolom | Sebelum | Sesudah |
|---|---|---|---|
| AST0101, AST0103 | status_asset | proses | **terpakai** |
| | pemegang_asset | gudang | **ruangan** |
| | id_pemegang_asset | `''` | **LOK0007** |
| | id_asset_lokasi_m | LOK0001 | **LOK0007** |
| AST0102 | status_asset | rusak | rusak (tidak berubah, tetap di gudang) |

`asset_trx_log` (2 baris; nomor contoh dengan asumsi ini log pertama pada periode 202610)

| id_asset_trx_log | id_asset_data | jenis_log | id_transaksi | nama_tabel_transaksi | kondisi awal → akhir | posisi awal → akhir | id_pemegang / nama_pemegang |
|---|---|---|---|---|---|---|---|
| CMP00120261000001 | AST0101 | onboarding_ruangan | CMP001202610001 | asset_trx_onboarding_ruangan | normal → normal | gudang → ruangan | LOK0007 / Ruang Meeting Lt.2 |
| CMP00120261000002 | AST0103 | onboarding_ruangan | CMP001202610001 | asset_trx_onboarding_ruangan | normal → normal | gudang → ruangan | LOK0007 / Ruang Meeting Lt.2 |

`asset_trx_onboarding_ruangan` (header)

| Kolom | Sebelum | Sesudah |
|---|---|---|
| status_approval | proses | **dikonfirmasi** |
| user_approval | `''` | ADM-001 - Siti Admin |
| tgl_approval | NULL | 2026-10-26 11:00:00 |

`notif_data`: baris untuk KRY0030, judul "Serah Terima Selesai", isi "Serah terima asset RGN01-202610001 telah dikonfirmasi dan selesai.".

**Diterima mobile** (HTTP 200)
```json
{ "code": "UPDATED", "message": "Alokasi asset ke ruangan berhasil diselesaikan.",
  "data": { "id": "CMP001202610001", "no_trx": "RGN01-202610001",
            "status_approval": "dikonfirmasi", "jumlah_asset": 2 } }
```
Transaksi pindah ke tab **Selesai**. `jumlah_asset = 2` menghitung aset aktif, bukan 3 baris item.

#### Kondisi akhir

| Objek | Keadaan |
|---|---|
| Header `CMP001202610001` | `dikonfirmasi`, 2 ronde, fase `selesai` |
| Item | 3 baris: Laptop (normal), Monitor LG (rusak, digantikan), Monitor Samsung (normal, ronde 2) |
| Jadwal | 3 baris: ronde 1 (dua versi), ronde 2 |
| AST0101, AST0103 | `terpakai`, pemegang `ruangan`, lokasi `LOK0007` |
| AST0102 | `rusak`, di gudang (menunggu penanganan perbaikan/disposal oleh alur lain) |
| `asset_lokasi_pic_m` | Dewi tercatat sebagai PIC tetap `LOK0007` |
| `asset_trx_log` | 2 baris `onboarding_ruangan` |

#### Variasi

- **Ruangan sudah punya PIC master:** field PIC terkunci di mobile dan `id_karyawan_pic` tidak perlu dikirim. Tidak ada INSERT ke `asset_lokasi_pic_m`; `id_asset_lokasi_pic_m` header menunjuk baris yang sudah ada. Mengirim PIC lain ditolak ("PIC Ruangan sudah ditetapkan pada ruangan ini dan tidak dapat diganti.").
- **Semua aset normal di ronde 1:** langkah 4 dan 5 dilewati; Admin langsung konfirmasi dari fase menunggu review. Transaksi hanya punya ronde 1.
- **Lebih dari satu aset rusak:** semuanya wajib diberi pengganti dalam satu permintaan; hasilnya satu baris item ronde berikutnya per aset rusak, masing-masing dengan `parent_id` sendiri.
- **Pengganti juga rusak:** ronde 3 dan seterusnya mengulang langkah 4–5 (`ronde + 1`, `parent_id` ke baris rusak terbaru); tidak ada batas ronde di kode.
- **Ubah jadwal berulang:** setiap perubahan menambah satu baris jadwal; baris terbaru per ronde adalah jadwal aktif.

#### Kasus gagal

Semua kesalahan aturan bisnis dilempar sebagai `InvalidArgumentException` di dalam transaksi, sehingga seluruh perubahan di-rollback dan controller membalas `BUSINESS_RULE_VIOLATION` dengan pesan di bawah. Tidak ada baris yang tersisa.

| Situasi | Langkah | Pesan |
|---|---|---|
| Pengguna bukan admin asset | 1, 2, 4, 6 | Pengguna tidak terdaftar sebagai admin asset. |
| Ruangan nonaktif atau bukan tipe ruangan | 1 | Ruangan tujuan tidak ditemukan atau tidak aktif. |
| Ruangan sudah punya transaksi `proses`/`dikonfirmasi` | 1 | Ruangan ini sudah pernah di-onboarding. Onboarding hanya dapat dilakukan satu kali per ruangan. |
| PIC master ada, admin mengirim PIC lain | 1 | PIC Ruangan sudah ditetapkan pada ruangan ini dan tidak dapat diganti. |
| Tanpa PIC master dan tanpa pilihan | 1 | PIC Ruangan belum ditetapkan untuk ruangan ini. |
| Aset sudah `proses` atau bukan di gudang | 1 | Satu atau beberapa asset tidak tersedia untuk dialokasikan. |
| Tanggal sebelum hari ini | 1, 2, 4 | Tanggal serah terima tidak boleh sebelum tanggal pengajuan. |
| Ubah jadwal setelah PIC menerima | 2 | Jadwal hanya dapat diubah sebelum PIC Ruangan menerima asset. |
| Transaksi sudah `dikonfirmasi` | 2, 4, 6 | Onboarding ruangan sudah selesai. |
| PIC lain atau ID salah | 3, 5 | Onboarding ruangan tidak ditemukan. |
| PIC mengirim checklist dua kali | 3, 5 | Onboarding ruangan sudah tidak menunggu serah terima. |
| Item checklist kurang atau tidak cocok | 3, 5 | Seluruh item pada serah terima wajib dicek. |
| Rusak tanpa keterangan | 3, 5 | Keterangan kerusakan wajib diisi. |
| Pengganti sebelum PIC mengirim checklist | 4 | Pengganti hanya dapat dijadwalkan setelah PIC Ruangan mengirim checklist. |
| Tidak ada aset rusak | 4 | Tidak ada asset rusak yang perlu diganti. |
| Pengganti tidak mencakup semua aset rusak | 4 | Seluruh asset rusak wajib diberi asset pengganti. |
| Pengganti tidak tersedia | 4 | Satu atau beberapa asset pengganti tidak tersedia. |
| Konfirmasi sebelum PIC mengirim checklist | 6 | Onboarding ruangan belum menunggu review Admin/HR. |
| Konfirmasi saat masih ada aset rusak | 6 | Masih ada asset rusak. Jadwalkan penggantian sebelum menyelesaikan alokasi. |
| Status aset berubah sejak dialokasikan | 6 | Satu atau beberapa asset tidak dapat diserahkan ke ruangan. |
| Nomor bulan ini sudah 999 | 1 | Nomor transaksi untuk periode ini sudah mencapai batas. |

#### Ringkasan siapa mengerjakan apa

| Pihak | Tugas dalam alur ini |
|---|---|
| **Mobile Admin/HR** | Memuat opsi, mengirim form alokasi/jadwal/pengganti, menampilkan timeline dan tombol sesuai `bisa_ubah_jadwal`, `perlu_pengganti`, `bisa_konfirmasi` |
| **Mobile PIC** | Menampilkan serah terima, mengumpulkan penilaian Normal/Rusak per aset, mengirim checklist |
| **API** | Validasi, penguncian company dan aset, penentuan PIC, penomoran, penurunan fase dari `tgl_terima_pic`, perubahan `asset_data`, pencatatan jadwal dan `asset_trx_log`, notifikasi bilingual |
| **Database** | Menyimpan header, item per ronde (rantai `parent_id`), riwayat jadwal append-only, master PIC ruangan, status aset, log, dan notifikasi |

## 46. Ringkasan Database Impact

Tabel baru `asset_trx_onboarding_ruangan_jadwal`; kolom `asset_lokasi_m.status_aktif`, `asset_trx_onboarding_ruangan.id_karyawan_pic`, `asset_trx_onboarding_ruangan_item.ronde`; header disamakan dengan KRY01 (enum status); `parent_id` item menjadi rantai pengganti; unique `asset_lokasi_pic_m (id_company, id_asset_lokasi_m)`.

## 47. Checklist Implementasi

### Database
- [x] DBML diperbarui
- [x] SQL manual + guard + migrasi + rollback
- [ ] SQL dijalankan di setiap environment (sudah di lingkungan pengembang)

### Backend
- [x] Model, controller admin dan PIC, route, notifikasi, test

### Mobile
- [x] Halaman PIC, form/daftar/detail admin, timeline, teks id/en, test

### Documentation
- [x] Dokumen ini
- [ ] Publikasi HTML/artifact dan kartu index

## 48. Changelog

| Versi | Tanggal | Perubahan |
|---|---|---|
| 1.0 | 13 Okt 2026 | Dokumen awal ALK-RGN01 (sisi PIC Ruangan dan Admin/HR) |
| 1.1 | 22 Okt 2026 | Menambah §45.4 walkthrough lengkap (mobile → payload → proses API → hasil database); koreksi nama master PIC menjadi `asset_lokasi_pic_m` / `id_asset_lokasi_pic_m` / `AssetLokasiPic_m`; log aset ditulis ke `asset_trx_log`. |
