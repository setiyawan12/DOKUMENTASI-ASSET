# Dokumentasi Teknis & Manual Book — Request / Penambahan Asset Karyawan

> Dokumen ini disusun **murni dari source code lokal** (`API/` Lumen dan `MOBILE/` Flutter) per 26 September 2026.
> Business requirement dan kode fitur **diturunkan dari kode**. Bagian yang tidak ditemukan ditandai **⚠️ Belum ditemukan**, asumsi ditandai **⚠️ Asumsi**.
> Contoh ID/nama pada payload adalah **data sintetis**.

---

## 1. Informasi Dokumen

| Informasi | Nilai |
|---|---|
| Nama Fitur | Request / Penambahan Asset Karyawan (UI: **"Penambahan Asset"** dan **"Pengajuan Asset"**) |
| Kode Fitur | **KRY-03-1** (prefix nomor transaksi `KRY-03-1-YYYYMMNNN`, dipakai kedua alur) |
| Modul | Asset Management |
| Backend | Laravel **Lumen ^10.0**, PHP ^8.1, JWT `tymon/jwt-auth ^2.3`, Query Builder (`DB::table`) |
| Database | Struktur mengacu `API/diagram-asset.dbml` + migrations. ⚠️ Asumsi: MySQL/MariaDB |
| Mobile | Flutter (package `professional_flutter_3_35`), `flutter_bloc`, `dio` |
| Web | ⚠️ Belum ditemukan: tidak ada source web di workspace |
| Versi Dokumen | 1.5 |
| Tanggal Dokumentasi | 2026-10-22 |
| Dokumen terkait | `dokumentasi-asset-onboarding-karyawan.md` (fitur serupa, banyak helper bersama); `sql-pengajuan-karyawan-jenis-asset-bebas.sql` (SQL manual v1.4, wajib dijalankan sebelum deploy API) |

---

## 2. Daftar Isi

1. Informasi Dokumen · 3. Quick Overview · 4. Ringkasan Fitur · 5. Tujuan · 6. Aktor · 7. Hak Akses · 8. Terminologi · 9. Business Rules · 10. Business Flow · 11. Flow per Role · 12. Flow Diagram · 13. Sequence Diagram · 14. Status Transaksi · 15. State Transition · 16. Struktur Database · 17. Relasi · 18. Data Dictionary · 19. Index · 20. Backend Architecture · 21. Backend Logic · 22. Dampak asset_data · 23. API · 24. Detail API · 25. Mobile Architecture · 26. Mobile Screen · 27. Mobile → API · 28. API → Mobile · 29. Mapping Mobile→API→DB · 30. Mapping DB→API→Mobile · 31. Validation · 32. Error Handling · 33. Audit Trail · 34. Notification · 35. Security · 36. Manual Book · 37. Kondisi Button · 38. Testing · 39. Temuan Ketidaksesuaian · 40. Edge Case · 41. Risk · 42. Dependency · 43. Source Reference · 44. Developer Notes · 45. Traceability · 46. DB Impact · 47. Checklist · 48. Changelog

---

## 3. Quick Overview

| Item | Informasi |
|---|---|
| Feature | Request / Penambahan Asset Karyawan |
| Module | Asset Management |
| Actor | Admin Asset, Karyawan, System |
| Main Transaction | `asset_trx_request_karyawan` |
| Detail Transaction | `asset_trx_request_karyawan_item` |
| Pembeda alur | Kolom `sumber_pengajuan`: `admin` = **Penambahan** (inisiatif admin), `karyawan` = **Pengajuan** (permintaan karyawan) |
| Asset Master | `asset_data` |
| Log | `asset_trx_log` (`jenis_log = request_karyawan`) |
| Initial Status | `draft` atau `proses` |
| Final Status | `dikonfirmasi` (UI: "Selesai") atau `ditolak` (hanya Pengajuan) |
| Approval | Hanya pada **Pengajuan**: Admin **Tolak** atau **Setujui (alokasi asset)** |
| API | 10 endpoint admin + 7 endpoint karyawan |
| Mobile Screen | Admin: Penambahan Asset (list, tambah, detail), Pengajuan Asset (list, detail, setujui). Karyawan: Pengajuan Asset (list, tambah, detail), Konfirmasi Penambahan/Pengajuan Asset, banner Asset Saya |

Satu tabel, dua pintu masuk, satu pintu keluar:

```text
PENAMBAHAN (sumber_pengajuan = admin)            PENGAJUAN (sumber_pengajuan = karyawan)
Admin: pilih karyawan + asset + jadwal            Karyawan: ketik jenis asset (teks) + alasan
        │ Submit ke Karyawan                              │ Submit ke Admin
        │ asset_data → proses                             ▼
        │                                         Admin: Tolak ──► status ditolak (selesai)
        │                                                 │ Setujui → alokasi asset + jadwal
        │                                                 │ Submit ke PIC → asset_data → proses
        ▼                                                 ▼
        └──────────────► Karyawan: Konfirmasi Asset (Normal / Rusak per item) ◄──┘
                                          │
              ┌── semua Normal ───────────┴───────── ada Rusak ──┐
              ▼                                                  ▼
  status dikonfirmasi ("Selesai")                     status tetap proses ("Butuh Proses")
  semua asset → terpakai, pemegang karyawan           TIDAK ada asset diserahkan
  + asset_trx_log                                         ⚠️ tidak ada mekanisme lanjutan (RISK-001)
```

---

## 4. Ringkasan Fitur

Fitur ini mencatat **pemberian asset tambahan kepada karyawan yang sudah bekerja** (berbeda dari Onboarding yang untuk penyerahan awal). Ada dua jalur yang disimpan di tabel yang sama:

- **Penambahan Asset (admin)** — Admin Asset langsung memilih karyawan, asset dari gudang (boleh lintas kategori), dan jadwal serah terima, lalu mengirim ke karyawan.
- **Pengajuan Asset (karyawan)** — Karyawan mengajukan **jenis asset** (teks bebas yang diketik sendiri, maks 150 karakter) beserta alasan. Admin meninjau: **menolak** dengan alasan, atau **menyetujui** dengan mengalokasikan asset fisik apa pun dari stok gudang (tidak dibatasi jenis yang diajukan) dan menentukan jadwal serah terima. Alokasi dapat disimpan sebagai draft terlebih dulu.

Kedua jalur berakhir di **konfirmasi karyawan**: tiap asset dinilai Normal atau Rusak. Berbeda dengan Onboarding, serah terima di fitur ini bersifat **all-or-nothing**: asset baru berpindah ke karyawan jika **semua** item Normal. Jika ada item Rusak, transaksi berhenti di status "Butuh Proses" dan **tidak ada endpoint lanjutan** di source.

---

## 5. Tujuan Fitur

### 5.1 Tujuan Bisnis (diturunkan dari kode)
- Karyawan dapat meminta asset tambahan secara tercatat dengan menuliskan jenis asset yang dibutuhkan (teks bebas) beserta alasan.
- Admin dapat menyetujui atau menolak permintaan dengan alasan yang tersimpan.
- Admin dapat memberikan asset tambahan tanpa menunggu permintaan.
- Penyerahan asset dijadwalkan (tanggal & jam) dan diverifikasi fisik oleh karyawan.
- Setiap perpindahan asset ke karyawan tercatat di `asset_trx_log`.

### 5.2 Tujuan Sistem
- Menyimpan pengajuan/penambahan dengan nomor `KRY-03-1-...`.
- Mengalokasikan asset fisik yang tersedia di gudang; admin bebas memilih asset apa pun dari stok (tidak dibatasi jenis yang diajukan).
- Mengunci asset (`status_asset = proses`) setelah dikirim ke karyawan.
- Memindahkan pemegang asset ke karyawan setelah konfirmasi normal.
- Mengirim notifikasi inbox + push ke pihak terkait.

---

## 6. Aktor

| Aktor | Role (di sistem) | Fungsi |
|---|---|---|
| Admin Asset | Terdaftar di `asset_admin` + karyawan aktif (`data_karyawan.status` ∈ '1'–'4') | Membuat penambahan; meninjau, menolak, dan mengalokasikan pengajuan |
| Karyawan | User login (JWT) | Membuat pengajuan; mengonfirmasi asset yang diterima |
| System | Backend + FCM | Validasi, lock, penomoran, update `asset_data`, `asset_trx_log`, notifikasi |

> Admin Asset berperan sebagai **approver** untuk Pengajuan. Tidak ada approver lain.

---

## 7. Hak Akses

| Action | Admin Asset | Karyawan | Keterangan |
|---|:---:|:---:|---|
| Melihat daftar Penambahan (semua) | ✅ | ❌ | `/asset/penambahan-karyawan` |
| Melihat daftar Pengajuan (semua, kecuali draft) | ✅ | ❌ | `/asset/pengajuan-karyawan` |
| Melihat pengajuan milik sendiri | – | ✅ | `/asset/karyawan/pengajuan` |
| Membuat Penambahan (draft/kirim) | ✅ | ❌ | |
| Membuat Pengajuan (draft/kirim) | – | ✅ | |
| Edit draft (simpan ulang / kirim) | ✅ (Penambahan) | ✅ (Pengajuan) | `PUT /asset/penambahan-karyawan/{id}`, `PUT /asset/karyawan/pengajuan/{id}`; hapus draft: ⚠️ belum ada (RISK-003) |
| Tolak pengajuan | ✅ (status Pengajuan) | ❌ | Wajib alasan |
| Setujui / alokasi asset | ✅ (status Pengajuan / Draft Alokasi) | ❌ | Asset apa pun dari stok gudang (tidak dibatasi jenis) |
| Konfirmasi asset (Normal/Rusak) | ❌ | ✅ (transaksi miliknya) | |
| Menindaklanjuti asset rusak | ❌ | ❌ | ⚠️ Belum ditemukan (RISK-001) |

---

## 8. Terminologi

| Istilah | Arti di sistem |
|---|---|
| Penambahan | Transaksi `sumber_pengajuan = admin` |
| Pengajuan | Transaksi `sumber_pengajuan = karyawan` |
| Jenis asset | **Teks bebas** (maks 150 karakter) yang diketik karyawan pada Pengajuan; disimpan di `nama_kategori`, tanpa relasi ke `asset_kategori_m` |
| Urgensi | **Dihapus** dari Pengajuan (v1.4). Kolom `urgensi` hanya legacy: baris lama tetap terbaca, tidak lagi diisi atau ditampilkan |
| Alokasi | Admin memilih asset fisik untuk pengajuan |
| Draft Alokasi | Alokasi tersimpan (`is_draft=true`): item ada, `tgl_approval` masih NULL |
| Submit ke PIC | Tombol kirim alokasi → pengajuan disetujui |
| `tgl_approval` | Pengajuan: diisi saat admin menyetujui, **ditimpa** saat karyawan konfirmasi. Penambahan: diisi saat karyawan konfirmasi |
| Pending item | Item `status_item = belum_dicek` |
| Butuh Proses | Label karyawan untuk transaksi yang memiliki item rusak |

---

## 9. Business Rules

### BR-001 — Endpoint admin hanya untuk Admin Asset
`Asset_c::accessError` (asset_admin + karyawan aktif) → 401/403; model memanggil `findAssetAdmin` → 422 "Pengguna tidak terdaftar sebagai admin asset."

### BR-002 — Penambahan: karyawan wajib dipilih & aktif
Validasi `id_karyawan` required; `findActiveKaryawan` → "Karyawan tidak ditemukan atau tidak aktif."

### BR-003 — Minimal satu asset, tanpa duplikat
`asset_ids` array min 1, distinct → "Minimal satu asset wajib dipilih." / "Asset yang sama tidak boleh dipilih lebih dari sekali."

### BR-004 — Asset harus tersedia di gudang
`requestLockedAssets`: `status_asset = tersedia` AND `pemegang_asset = gudang` (FOR UPDATE) → "Satu atau beberapa asset tidak tersedia untuk penambahan." / "Satu atau beberapa asset tidak tersedia." (alokasi).

### BR-005 — Asset tidak boleh aktif di onboarding atau penambahan lain
`hasActiveOnboardingAsset` → "…sedang diproses pada onboarding." `hasActiveRequestAsset` (header **`proses` saja**, sumber admin atau pengajuan yang sudah disetujui) → "…sedang diproses pada penambahan lain." Transaksi `dikonfirmasi` (selesai) **tidak lagi** mengunci asset. Asset yang masih dipegang karyawan tetap tidak bisa dipilih karena syarat `status_asset = tersedia` + `pemegang_asset = gudang`, sehingga asset hasil Retur/Offboarding bisa ditambahkan lagi (RISK-002, sudah diperbaiki v1.1).

### BR-006 — Jadwal serah terima wajib saat dikirim
Penambahan & alokasi: `tgl_serah_terima` (Y-m-d) dan `jam_serah_terima` (H:i) required bila `is_draft=false`. Draft boleh kosong (disimpan `0000-00-00` / `00:00:00`). Backend **tidak** memvalidasi tanggal ≥ hari ini (mobile membatasi `firstDate = hari ini`).

### BR-007 — Pengajuan: jenis asset wajib (teks bebas)
`jenis_asset` (string, di-trim, maks 150) required bila bukan draft; draft boleh kosong. Tidak ada validasi terhadap `asset_kategori_m`. Disimpan ke `nama_kategori`; `id_asset_kategori_m` dan `urgensi` diisi NULL. Field `id_asset_kategori_m` dan `urgensi` dari klien lama diabaikan → "Jenis asset wajib diisi." (422 `VALIDATION_ERROR`; bila ada field lain yang gagal, message induk "Data pengajuan asset belum lengkap.")

### BR-008 — Pengajuan: alasan wajib saat dikirim
`keterangan` required bila bukan draft → "Alasan pengajuan wajib diisi." Urgensi **tidak ada lagi** pada Pengajuan (dihapus v1.4): tidak ada form, validasi, maupun tampilan.

### BR-009 — Tolak/alokasi hanya saat pengajuan belum disetujui
Syarat: `sumber_pengajuan = karyawan`, `status_approval = proses`, `tgl_approval IS NULL` → "Pengajuan asset sudah tidak menunggu persetujuan." / "…sudah tidak dapat dialokasikan."

### BR-010 — Alasan penolakan wajib (maks 255)
→ "Alasan penolakan wajib diisi."

### BR-011 — Alokasi bebas lintas jenis (v1.4)
Admin dapat memilih asset **apa pun** dari stok gudang. Jenis yang diajukan karyawan (teks bebas) hanya informasi dan tidak dibandingkan dengan `asset_data.id_asset_kategori_m`; pengecekan "Asset harus sesuai dengan jenis yang diajukan." dihapus dan options alokasi tidak lagi memfilter kategori. Validasi lain tetap: tersedia + di gudang (BR-004), tidak aktif di transaksi lain (BR-005), status pengajuan (BR-009), otorisasi admin (BR-001), serta lock/transaksi.

### BR-012 — Alokasi dapat disimpan sebagai draft dan diulang
Setiap alokasi menghapus item lama lalu menulis item baru. Draft **tidak** mengunci asset dan **tidak** mengisi `tgl_approval`.

### BR-013 — Karyawan hanya melihat & mengonfirmasi transaksi miliknya yang sudah dikirim
Detail/konfirmasi memfilter `id_karyawan = ctx.id_karyawan`, bukan draft, dan untuk pengajuan harus sudah disetujui → "Pengajuan asset belum disubmit oleh admin."

### BR-014 — Semua item pending wajib dinilai; rusak wajib alasan
→ "Seluruh asset penambahan wajib dicek." / "Alasan kerusakan wajib diisi." / "Tidak ada asset penambahan yang perlu dicek."

### BR-015 — Serah terima all-or-nothing
Asset baru diserahkan (`terpakai`, pemegang karyawan, `asset_trx_log`) **hanya jika seluruh item Normal**. Jika ada Rusak, status tetap `proses` dan **tidak ada asset** yang berubah.

### BR-016 — Tidak ada tindak lanjut asset rusak
⚠️ Belum ditemukan: endpoint ganti/tarik/batal untuk transaksi request. Lihat RISK-001.

### BR-017 — Penomoran
ID header & item: `id_company + YYYYMM + 3 digit` (maks 999/bulan/company, dipakai bersama Penambahan & Pengajuan). `no_trx = KRY-03-1-YYYYMMNNN`.

---

## 10. Business Flow

### 10.1 Penambahan (Admin → Karyawan)
```text
Admin: Asset Karyawan → Penambahan → (+)
  → Pilih Karyawan → Tambah Asset (boleh multi kategori) → Jadwal Serah Terima (tgl + jam) → Catatan Tambahan
  → Simpan Draft  : status draft, asset TIDAK dikunci; dibuka lagi lewat Ubah Draft lalu disimpan/dikirim (PUT)
  → Submit ke Karyawan : status proses, asset_data → proses, notif ke karyawan
Karyawan: Asset Saya → banner "Penambahan Asset" → Konfirmasi Penambahan Asset
  → Normal/Rusak tiap asset → Konfirmasi Asset
  → semua Normal : status dikonfirmasi, asset → terpakai/karyawan, asset_trx_log, notif admin
  → ada Rusak    : status proses (tgl_approval terisi), tidak ada perubahan asset, notif admin "Laporan Asset Rusak"
```

### 10.2 Pengajuan (Karyawan → Admin → Karyawan)
```text
Karyawan: Asset Saya → Pengajuan → (+)
  → Jenis Asset (ketik bebas) → Alasan → Simpan Draft / Submit ke Admin
  → Submit : status proses, tgl_approval NULL, tanpa item, notif ke semua admin
Admin: Asset Karyawan → Pengajuan (tab Pengajuan) → Detail
  → Tolak  : isi alasan → status ditolak, item dihapus, notif karyawan
  → Setujui: pilih asset apa pun dari stok + jadwal + catatan
       → Simpan Draft : "Draft Alokasi" (item tersimpan, asset belum dikunci)
       → Submit ke PIC: tgl_approval terisi, asset_data → proses, notif karyawan "Disetujui"
Karyawan: Pengajuan (tab Dikonfirmasi, "Menunggu Konfirmasi") → Konfirmasi Pengajuan Asset
  → sama seperti konfirmasi Penambahan
```

---

## 11. Flow Berdasarkan Role

### 11.1 Admin Asset
1. Buka **Asset Karyawan** → aksi **Penambahan** atau **Pengajuan** (juga panel "Pengajuan Asset Perlu Diproses" berisi maks. 3 pengajuan tertua, tombol "Tinjau Pengajuan" / "Lanjutkan Alokasi").
2. **Penambahan**: tab Pending / Dikonfirmasi / Selesai; tombol + untuk membuat; tap kartu → detail (read-only).
3. **Pengajuan**: tab Pengajuan / Proses / Dikonfirmasi / Selesai; tap kartu → **Konfirmasi Pengajuan Asset** (detail) → **Tolak** atau **Setujui** → form **Setujui Pengajuan Asset** → Simpan Draft / Submit ke PIC.

### 11.2 Karyawan
1. **Asset Saya** → menu **Pengajuan** → daftar pengajuan (tab Pengajuan / Dikonfirmasi / Ditolak / Selesai) → + untuk membuat.
2. Pengajuan yang sudah dialokasikan & menunggu dicek dibuka langsung ke **Konfirmasi Pengajuan Asset**.
3. Penambahan dari admin muncul sebagai banner **Penambahan Asset** di Asset Saya (hanya yang pertama) dan via notifikasi.

### 11.3 Approver
Peran approver dijalankan Admin Asset pada Pengajuan (§11.1 langkah 3).

### 11.4 System
Lock `master_company`, generate ID/nomor, validasi asset & kategori, reservasi asset saat dikirim, penentuan status/label/tab, serah terima + `asset_trx_log` saat semua normal, notifikasi setelah commit.

---

## 12. Flow Diagram

```mermaid
flowchart TD
    subgraph P[Penambahan - Admin]
        A1[Admin isi form penambahan] --> A2{Aksi}
        A2 -->|Simpan Draft| A3[status draft - bisa diubah lalu dikirim (PUT)]
        A2 -->|Submit ke Karyawan| A4[status proses, asset proses, notif karyawan]
    end
    subgraph Q[Pengajuan - Karyawan]
        B1[Karyawan ketik jenis asset dan alasan] --> B2{Aksi}
        B2 -->|Simpan Draft| B3[status draft - bisa diubah lalu dikirim (PUT)]
        B2 -->|Submit ke Admin| B4[status proses, tanpa item, notif admin]
        B4 --> B5{Keputusan admin}
        B5 -->|Tolak + alasan| B6[status ditolak, notif karyawan]
        B5 -->|Setujui| B7[Alokasi asset dari stok + jadwal]
        B7 -->|Simpan Draft| B8[Draft Alokasi]
        B8 --> B5
        B7 -->|Submit ke PIC| B9[tgl_approval terisi, asset proses, notif karyawan]
    end
    A4 --> C1[Karyawan: Konfirmasi Asset]
    B9 --> C1
    C1 --> C2{Semua item Normal?}
    C2 -->|Ya| C3[status dikonfirmasi - Selesai, asset terpakai, asset_trx_log, notif admin]
    C2 -->|Tidak| C4[status proses - Butuh Proses, asset tetap proses, notif admin]
    C4 --> C5[Tidak ada tindak lanjut di sistem]
```

---

## 13. Sequence Diagram

### 13.1 Pengajuan karyawan sampai disetujui

```mermaid
sequenceDiagram
    actor K as Karyawan
    actor A as Admin
    participant M as Mobile
    participant CK as AssetKaryawanPengajuan_c
    participant CA as AssetAdminPenambahan_c
    participant R as AssetRequestKaryawan_m
    participant DB
    participant N as AssetNotification

    K->>M: Ketik jenis asset dan alasan, Submit ke Admin
    M->>CK: POST /asset/karyawan/pengajuan
    CK->>R: storePengajuanKaryawan()
    R->>DB: lock company, cek karyawan aktif
    R->>DB: INSERT header (sumber karyawan, proses, nama_kategori = jenis asset)
    CK->>N: pengajuan.diajukan (semua admin)
    A->>M: Buka detail, Setujui, pilih asset + jadwal
    M->>CA: GET /asset/pengajuan-karyawan/{id}/options
    M->>CA: POST /asset/pengajuan-karyawan/{id}/alokasi (is_draft false)
    CA->>R: alokasikanPengajuanAdmin()
    R->>DB: validasi asset tersedia di gudang, tidak aktif, tanpa filter kategori
    R->>DB: DELETE item lama, INSERT item baru
    R->>DB: UPDATE header jadwal, user_approval, tgl_approval
    R->>DB: UPDATE asset_data status proses
    CA->>N: pengajuan.disetujui (karyawan)
    CA-->>M: 200 SUCCESS
```

### 13.2 Konfirmasi karyawan (kedua alur)

```mermaid
sequenceDiagram
    actor K as Karyawan
    participant M as Konfirmasi Penambahan/Pengajuan
    participant C as AssetKaryawanPengajuan_c
    participant R as AssetRequestKaryawan_m
    participant DB
    participant N as AssetNotification

    K->>M: Nilai Normal/Rusak tiap asset, Konfirmasi Asset
    M->>C: POST /asset/karyawan/penambahan/{id}/konfirmasi
    C->>R: konfirmasiKaryawan()
    R->>DB: lock company, header milik karyawan FOR UPDATE
    R->>DB: UPDATE item status_item, ket_rusak
    R->>DB: UPDATE header status, user_approval, tgl_approval
    alt semua item normal
        R->>DB: UPDATE asset_data terpakai, pemegang karyawan
        R->>DB: INSERT asset_trx_log request_karyawan
        C->>N: penambahan.dikonfirmasi (admin)
    else ada item rusak
        C->>N: penambahan.rusak (admin)
    end
    C-->>M: 200 UPDATED
```

---

## 14. Status Transaksi

Field `asset_trx_request_karyawan.status_approval` (enum `status_request_karyawan`: `draft`, `proses`, `ditolak`, `dikonfirmasi`; default `proses`). Label & tab **dihitung** dari status + `tgl_approval` + item, dan berbeda per sudut pandang.

### 14.1 Penambahan — tampilan Admin (`requestStatusMetadata`)
| Kondisi | Label | Tab |
|---|---|---|
| `draft` | Draft | pending |
| `proses`, `tgl_approval` NULL | Menunggu Diterima | pending |
| `proses`, `tgl_approval` terisi (karyawan melaporkan rusak) | Dikonfirmasi | dikonfirmasi |
| `dikonfirmasi` | Selesai | selesai |
| `ditolak` | Ditolak | selesai (tidak terjadi pada Penambahan) |

### 14.2 Pengajuan — tampilan Admin (`pengajuanAdminStatusMetadata`, draft tidak ditampilkan)
| Kondisi | Label | Tab |
|---|---|---|
| `proses`, belum disetujui, tanpa item | Menunggu Diterima | pengajuan |
| `proses`, belum disetujui, ada item | Draft Alokasi | pengajuan |
| `proses`, disetujui, ada item `belum_dicek` | Menunggu Konfirmasi | proses |
| `proses`, disetujui, tanpa item pending (ada rusak) | Dikonfirmasi | dikonfirmasi |
| `ditolak` | Ditolak | selesai |
| `dikonfirmasi` | Selesai | selesai |

### 14.3 Pengajuan — tampilan Karyawan (`pengajuanStatusMetadata`)
| Kondisi | Label | Tab |
|---|---|---|
| `draft` | Draft | pengajuan |
| `proses`, belum disetujui | Menunggu Konfirmasi | pengajuan |
| `proses`, disetujui, ada item rusak | Butuh Proses | dikonfirmasi |
| `proses`, disetujui, ada item pending | Menunggu Konfirmasi | dikonfirmasi |
| `proses`, disetujui, lainnya | Dikonfirmasi Admin | dikonfirmasi |
| `ditolak` | Ditolak | ditolak |
| `dikonfirmasi` | Selesai | selesai |

Status item (`status_item`): `belum_dicek` → `normal` / `rusak`.
Migrasi `2026_09_17_000008`: nilai lama 0=draft, 1=proses, 2=ditolak, 3=dikonfirmasi.

---

## 15. State Transition

```mermaid
stateDiagram-v2
    [*] --> draft: Simpan Draft (Admin / Karyawan)
    [*] --> proses: Submit (Admin / Karyawan)
    state proses {
        MenungguAdmin --> DraftAlokasi: Alokasi draft (Admin)
        DraftAlokasi --> DraftAlokasi: Alokasi draft ulang (Admin)
        MenungguAdmin --> MenungguKaryawan: Submit ke PIC (Admin)
        DraftAlokasi --> MenungguKaryawan: Submit ke PIC (Admin)
        MenungguKaryawan --> ButuhProses: Konfirmasi ada rusak (Karyawan)
    }
    proses --> ditolak: Tolak (Admin, hanya pengajuan belum disetujui)
    proses --> dikonfirmasi: Konfirmasi semua normal (Karyawan)
    ditolak --> [*]
    dikonfirmasi --> [*]
```

> Pada Penambahan, transaksi langsung berada di tahap `MenungguKaryawan` setelah Submit ke Karyawan. Keadaan `ButuhProses` tidak memiliki transisi keluar; `draft` dapat disimpan ulang atau dikirim lewat Edit Draft (PUT).

| Status Awal | Action | Status Akhir | Actor |
|---|---|---|---|
| – | Simpan Draft | draft | Admin / Karyawan |
| – | Submit ke Karyawan (penambahan) | proses (menunggu karyawan) | Admin |
| – | Submit ke Admin (pengajuan) | proses (menunggu admin) | Karyawan |
| draft | Edit Draft: simpan ulang (`is_draft=true`) | draft | Admin (penambahan) / Karyawan (pengajuan) |
| draft | Edit Draft: kirim (`is_draft=false`) | proses (menunggu karyawan / menunggu admin) | Admin (penambahan) / Karyawan (pengajuan) |
| proses (menunggu admin) | Tolak | ditolak | Admin |
| proses (menunggu admin / draft alokasi) | Alokasi `is_draft=true` | proses (Draft Alokasi) | Admin |
| proses (menunggu admin / draft alokasi) | Alokasi `is_draft=false` | proses (menunggu karyawan) | Admin |
| proses (menunggu karyawan) | Konfirmasi semua normal | dikonfirmasi | Karyawan |
| proses (menunggu karyawan) | Konfirmasi ada rusak | proses (Butuh Proses) | Karyawan |
| Butuh Proses | – | ⚠️ tidak ada transisi | – |

Status `asset_data.status_asset`:
| Dari | Ke | Trigger |
|---|---|---|
| tersedia (gudang) | proses | Submit penambahan; Submit alokasi (bukan draft) |
| proses (gudang) | terpakai (karyawan) | Konfirmasi dengan **semua** item normal |
| proses | proses (tetap) | Konfirmasi dengan item rusak — **tidak berubah** |

---

## 16. Struktur Database

| Table | Type | Fungsi |
|---|---|---|
| `asset_trx_request_karyawan` | Transaction Header | Penambahan & Pengajuan |
| `asset_trx_request_karyawan_item` | Transaction Detail | Asset yang dialokasikan |
| `asset_data` | Master | Status & pemegang asset |
| `asset_kategori_m` | Master | Snapshot kategori asset (Penambahan) dan pohon pilihan asset; **bukan lagi** sumber jenis asset Pengajuan |
| `data_karyawan`, `master_cabang`, `master_departemen`, `master_jabatan` | Master HR | Karyawan aktif, jabatan, cabang, departemen |
| `asset_admin` | Master akses | Admin asset |
| `master_company` | Master | Lock transaksi |
| `asset_trx_log` | Log | Histori serah terima (`request_karyawan`) |
| `notif_data`, `users` | Notifikasi | Inbox & token FCM |

> ⚠️ Belum ditemukan: migration `CREATE TABLE` untuk tabel request; struktur mengacu DBML + migrations `2026_09_16_000003`, `2026_09_17_000004`, `2026_09_17_000008`.

> **Perubahan skema v1.4 — SQL manual, BELUM dijalankan:** `Claude outputs/sql-pengajuan-karyawan-jenis-asset-bebas.sql` mengubah `asset_trx_request_karyawan.id_asset_kategori_m` dan `urgensi` menjadi NULL diperbolehkan (tipe tetap `varchar(50)`); data lama tidak diubah. `API/diagram-asset.dbml` sudah ditandai nullable. **Jalankan SQL ini di database sebelum API v1.4 dipakai**; tanpa itu INSERT Pengajuan baru (nilai NULL pada `id_asset_kategori_m`) gagal. Script memuat peringatan cek charset/collation (`SHOW CREATE TABLE`) karena `nama_kategori` kini menampung teks bebas (latin1 vs utf8mb4); script tidak mengubah charset.

---

## 17. Relasi Database

```mermaid
erDiagram
    ASSET_TRX_REQUEST_KARYAWAN ||--o{ ASSET_TRX_REQUEST_KARYAWAN_ITEM : "id_asset_trx_request"
    ASSET_DATA ||--o{ ASSET_TRX_REQUEST_KARYAWAN_ITEM : "id_asset_data"
    ASSET_KATEGORI_M |o--o{ ASSET_TRX_REQUEST_KARYAWAN : "id_asset_kategori_m (legacy, nullable)"
    ASSET_KATEGORI_M ||--o{ ASSET_DATA : "id_asset_kategori_m"
    DATA_KARYAWAN ||--o{ ASSET_TRX_REQUEST_KARYAWAN : "id_karyawan (logis)"
    ASSET_DATA ||--o{ ASSET_TRX_LOG : "id_asset_data"
    ASSET_TRX_REQUEST_KARYAWAN ||--o{ ASSET_TRX_LOG : "id_transaksi (logis)"

    ASSET_TRX_REQUEST_KARYAWAN {
        varchar id_asset_trx_request_karyawan PK
        varchar no_trx
        enum sumber_pengajuan
        varchar id_asset_kategori_m "nullable, legacy"
        varchar nama_kategori "jenis asset teks bebas"
        varchar urgensi "nullable, legacy"
        varchar id_karyawan
        date tgl_serah_terima
        enum status_approval
        timestamp tgl_approval
        varchar id_company
    }
    ASSET_TRX_REQUEST_KARYAWAN_ITEM {
        varchar id_asset_trx_request_item PK
        varchar id_asset_trx_request FK
        varchar id_asset_data FK
        enum status_item
        varchar ket_rusak
    }
    ASSET_DATA {
        varchar id_asset_data PK
        varchar id_asset_kategori_m FK
        enum status_asset
        enum pemegang_asset
        varchar id_pemegang_asset
    }
    ASSET_TRX_LOG {
        varchar id_asset_trx_log PK
        varchar id_asset_data FK
        enum jenis_log
        varchar id_transaksi
    }
```

Relasi logis (tanpa `ref` DBML): `id_karyawan` → `data_karyawan`, `id_asset_kategori_m` (header, legacy/nullable) → `asset_kategori_m`, `asset_trx_log.id_transaksi` → header. Semua JOIN mencocokkan `id_company`. Kolom item `parent_id` ada di DBML tetapi selalu `''` (tidak ada penggantian).

---

## 18. Data Dictionary

### 18.1 asset_trx_request_karyawan
| Field | Type | Null | Default | Key | Description |
|---|---|:---:|---|---|---|
| id_asset_trx_request_karyawan | varchar(50) | No | – | PK | `id_company + YYYYMM + 3 digit` |
| no_trx | varchar(50) | No | '' | UNIQUE (DBML) | `KRY-03-1-YYYYMMNNN` |
| sumber_pengajuan | enum(admin, karyawan) | No | admin | IDX | `admin` = Penambahan, `karyawan` = Pengajuan |
| id_asset_kategori_m | varchar(50) | Yes (setelah SQL v1.4) | NULL | – | Penambahan: kategori bila semua asset satu kategori, selain itu `''`. **Pengajuan baru: NULL** (legacy; baris lama tetap berisi kategori). API membaca NULL sebagai `''` |
| nama_kategori | varchar(255) | No | '' | – | Penambahan: snapshot kategori; multi kategori = `Multi Kategori`. **Pengajuan: teks jenis asset bebas** (maks 150 karakter dari API) |
| keterangan | text | Yes | – | – | Pengajuan: alasan karyawan. Penambahan: selalu `''` dari mobile |
| urgensi | varchar(50) | Yes | NULL | – | **LEGACY**: tidak dipakai lagi pada Pengajuan baru (selalu NULL). Baris lama berisi `normal`/`secepatnya`/`mendesak` dan tetap tersimpan; Penambahan admin menulis `''`. Tidak ditampilkan di API/mobile |
| ket_tolak | varchar(255) | No | '' | – | Alasan penolakan admin |
| id_karyawan, nik_karyawan, nama_karyawan | varchar | – | '' | IDX | Karyawan penerima/pemohon (snapshot) |
| tgl_pengajuan | date | No | CURRENT_TIMESTAMP | – | Tanggal dibuat |
| tgl_serah_terima | date | No | 0000-00-00 | – | Jadwal serah terima |
| jam_serah_terima | time | No | 00:00:00 | – | Jam serah terima (disimpan H:i:s) |
| keterangan_serah_terima | varchar(255) | – | '' | – | "Catatan Tambahan" pada form admin |
| status_approval | enum(draft, proses, ditolak, dikonfirmasi) | No | proses | IDX | Status |
| user_approval | varchar(255) | No | '' | – | Admin penyetuju/penolak, lalu **ditimpa** karyawan saat konfirmasi |
| tgl_approval | timestamp | Yes | NULL | IDX | Lihat §8 |
| id_company | varchar(50) | No | '' | IDX | Company |
| tgl_input, user_input | – | – | – | IDX | Pembuat (`nik - nama`) |
| tgl_update, user_update | – | Yes | NULL | – | Pengubah terakhir |

### 18.2 asset_trx_request_karyawan_item
| Field | Type | Null | Default | Key | Description |
|---|---|:---:|---|---|---|
| id_asset_trx_request_item | varchar(50) | No | – | PK | `id_company + YYYYMM + 3 digit` |
| parent_id | varchar(50) | Yes | – | – | Selalu `''` |
| id_asset_trx_request | varchar(50) | – | – | FK, IDX | → header |
| id_asset_data | varchar(50) | – | – | FK | → asset_data |
| nama_asset, deskripsi_asset, kategori_asset | varchar(255) | – | '' | – | Snapshot asset (API: `nama_asset`, `deskripsi`, `kategori_asset`) |
| status_item | enum(normal, rusak, belum_dicek) | – | – | IDX | Awal `belum_dicek` |
| ket_rusak | varchar(255) | – | '' | – | Alasan rusak dari karyawan |
| id_company, tgl_input, user_input | – | – | – | IDX | – |

### 18.3 asset_data (field yang dipakai)
`id_asset_data`, `id_asset_kategori_m` (snapshot kategori; tidak lagi dipakai membatasi alokasi), `nama_asset`, `deskripsi`, `kode_asset`, `merek`, `status_asset` (tersedia → proses → terpakai), `pemegang_asset` (gudang → karyawan), `id_pemegang_asset`, `kondisi_awal` (disalin ke log), `tgl_update`, `user_update`.

### 18.4 asset_trx_log (diisi saat semua normal)
`jenis_log = request_karyawan`, `id_transaksi` = ID header, `nama_tabel_transaksi = asset_trx_request_karyawan`, `kondisi_awal` = `asset_data.kondisi_awal`, `kondisi_akhir = normal`, `posisi_awal` = pemegang sebelumnya (`gudang`), `posisi_akhir = pemegang_asset = karyawan`, `id_pemegang`, `nama_pemegang`, `user_input` = karyawan.

---

## 19. Database Index

| Index | Table | Columns | Tujuan |
|---|---|---|---|
| `atrk_company_source_employee_idx` | header | (id_company, sumber_pengajuan, id_karyawan) | Filter per sumber & karyawan |
| `atrk_admin_list_idx` | header | (id_company, sumber_pengajuan, status_approval, tgl_approval, tgl_input, id_asset_trx_request_karyawan) | List admin per tab + urutan |
| `atrk_employee_list_idx` | header | (id_company, sumber_pengajuan, id_karyawan, status_approval, tgl_approval, tgl_input, id_asset_trx_request_karyawan) | List pengajuan karyawan |
| `atrki_request_status_idx` | item | (id_asset_trx_request, id_company, status_item) | COUNT item / pending / rusak per header |
| UNIQUE `no_trx` | header | (no_trx) | DBML — ⚠️ belum diverifikasi ada di DB fisik |

Pencarian `hasActiveRequestAsset` berdasarkan `atrki.id_asset_data` tidak diawali kolom index yang ada (potensi lambat, ⚠️ belum diukur).

---

## 20. Backend Architecture

```text
routes/modules/asset/RouteAsset.php (prefix /asset, middleware jwt.api + jwt.ctx)
   ↓
AssetAdminPenambahan_c / AssetKaryawanPengajuan_c  (validateRequest inline, accessError)
   ↓
AssetRequestKaryawan_m  ← extends Asset_m (lock, sequence, findActiveKaryawan, findAssetAdmin, hasActiveOnboardingAsset)
   ↓                    ← memakai AssetAdmin_m::getOnboardingOptions untuk opsi penambahan/alokasi
DB (Query Builder, DB::transaction retry 3)
   ↓
ApiResponse + AssetNotification (setelah commit)
```

| Layer | File/Class | Function | Tanggung Jawab |
|---|---|---|---|
| Route | `RouteAsset.php` | 17 route | Endpoint |
| Controller | `AssetAdminPenambahan_c` / `AssetAdminPengajuan_c` | penambahanOptions, getPenambahanKaryawan, storePenambahanKaryawan, getDetailPenambahanKaryawan, getPengajuanKaryawan, getDetailPengajuanKaryawan, pengajuanKaryawanOptions, tolakPengajuanKaryawan, alokasikanPengajuanKaryawan | Admin |
| Controller | `AssetKaryawanPengajuan_c` / `AssetKaryawanPenambahan_c` | getPengajuanOptionsSaya, getPengajuanSaya, storePengajuanSaya, getDetailPengajuanSaya, getDetailPenambahanSaya, konfirmasiPenambahanSaya | Karyawan |
| Model | `AssetRequestKaryawan_m` | getOptions, getPengajuanOptions, getRequests/Page, getPengajuanAdmin/Page, getPengajuanKaryawan/Page, getDetail, getDetailPengajuanAdmin, getDetailPengajuanKaryawan, getPengajuanAdminOptions, store, storePengajuanKaryawan, tolakPengajuanAdmin, alokasikanPengajuanAdmin, konfirmasiKaryawan, getPendingForEmployee, getPendingForAdmin, countPendingForAdmin | Logic |
| Model | `Asset_m` | lockCompany, nextSequence, ensureSequenceCapacity, findActiveKaryawan, findAssetAdmin, hasActiveOnboardingAsset | Helper bersama |
| Enum | `RequestKaryawanStatus` | – | Ada, tetapi model memakai konstanta string sendiri |
| Support | `AssetNotification` | send | Notifikasi |

---

## 21. Backend Business Logic

Semua aksi tulis: `DB::transaction(fn, 3)` + `lockCompany()`.

### 21.1 Penambahan — `store`
1. Validasi request → cek admin.
2. Lock; `requestLockedAssets` (tersedia + gudang, FOR UPDATE), jumlah harus sama.
3. `hasActiveOnboardingAsset`, `hasActiveRequestAsset`.
4. `findActiveKaryawan`, `findAssetAdmin`.
5. Jadwal wajib bila bukan draft.
6. Generate ID + `no_trx = KRY-03-1-YYYYMMNNN`.
7. Tentukan kategori: satu kategori → id & nama; banyak → `''` & "Multi Kategori".
8. INSERT header (`sumber_pengajuan = admin`, draft/proses, `tgl_approval` NULL).
9. INSERT item (`belum_dicek`, snapshot).
10. **Hanya jika bukan draft**: UPDATE `asset_data.status_asset = proses`.
11. Notifikasi `penambahan.dikirim` (bukan draft). Response 201.

### 21.2 Pengajuan — `storePengajuanKaryawan`
Lock → karyawan aktif → `jenis_asset` (di-trim) dan alasan wajib bila bukan draft → generate ID/no_trx → INSERT header (`sumber = karyawan`, `nama_kategori = jenis_asset`, `id_asset_kategori_m = NULL`, `urgensi = NULL`, tanpa item, jadwal kosong) → notifikasi `pengajuan.diajukan` (bukan draft). Response 201.

### 21.3 Tolak — `tolakPengajuanAdmin`
Lock → admin → header (sumber karyawan, FOR UPDATE) harus `proses` & `tgl_approval` NULL → alasan wajib → **DELETE item** (draft alokasi) → UPDATE header `ket_tolak`, `status_approval = ditolak`, `user_approval`, `tgl_approval = now` → notifikasi `pengajuan.ditolak`.

### 21.4 Alokasi — `alokasikanPengajuanAdmin`
Lock → admin → header syarat sama dengan tolak → asset tersedia di gudang (tanpa filter kategori; jenis asset pengajuan hanya informasi) → tidak aktif di onboarding/penambahan lain (kecuali transaksi ini) → jadwal wajib bila bukan draft → DELETE item lama → INSERT item baru → UPDATE header jadwal, `ket_tolak = ''`, `user_approval`/`tgl_approval` (hanya bila bukan draft) → bila bukan draft UPDATE `asset_data = proses` → notifikasi `pengajuan.disetujui` (bukan draft). Response 200 `SUCCESS` (kirim) / `UPDATED` (draft).

### 21.5 Konfirmasi — `konfirmasiKaryawan`
1. Lock; header milik karyawan (FOR UPDATE), status `proses`; pengajuan harus sudah disetujui.
2. Karyawan aktif; item dikunci; pending = `belum_dicek`.
3. Validasi lengkap & alasan rusak.
4. UPDATE item.
5. `semuaNormal` atas **seluruh** item → `dikonfirmasi`, selain itu tetap `proses`.
6. UPDATE header `status_approval`, `user_approval`, `tgl_approval = now`.
7. **Jika semua normal**: UPDATE `asset_data` terpakai/karyawan/id_pemegang_asset + INSERT `asset_trx_log` per asset.
8. **Jika ada rusak**: tidak ada perubahan `asset_data` (tetap `proses`).
9. Notifikasi `penambahan.dikonfirmasi` / `penambahan.rusak`.

### 21.6 Query baca
- Options penambahan: `AssetAdmin_m::getOnboardingOptions(excludeEmployeesWithOnboarding: false)` → semua karyawan aktif, asset tersedia di gudang, pohon kategori.
- Options alokasi: sama (seluruh stok tersedia di gudang), **tanpa filter kategori** (v1.4). Hanya bila pengajuan `proses` & belum disetujui (selain itu 404).
- Options pengajuan karyawan (legacy): hanya `kategori_asset` (semua kategori company); daftar `urgensi` dihapus. Mobile v1.4 tidak lagi memanggil endpoint ini.
- List: JOIN item, hitung `jumlah_asset`, `jumlah_pending`, `jumlah_rusak`; urut `tgl_input DESC`.
- Detail karyawan pengajuan: sebelum disetujui, jadwal dikosongkan, `jumlah_asset = 0`, `items = []`.
- Detail (`getDetail`): objek `karyawan` memuat `jabatan` (LEFT JOIN `master_jabatan` lewat `data_karyawan.id_jabatan`; `''` bila kosong) selain `cabang` dan `divisi`.
- Response list/detail/pending Pengajuan: `nama_kategori` dan `nama_asset` = teks jenis asset; `id_asset_kategori_m` = `''` untuk data baru; tanpa `urgensi` (termasuk detail Penambahan admin yang berbagi query; penulisan `urgensi` pada `store()` Penambahan admin tetap ada, nullable).

---

## 22. Dampak terhadap asset_data

| Field | Sebelum | Sesudah | Trigger |
|---|---|---|---|
| status_asset | tersedia | proses | Submit penambahan / submit alokasi |
| status_asset | proses | terpakai | Konfirmasi, **semua** item normal |
| pemegang_asset | gudang | karyawan | Konfirmasi, semua normal |
| id_pemegang_asset | '' | id_karyawan | Konfirmasi, semua normal |
| id_asset_lokasi_m | lokasi gudang | NULL | Konfirmasi, semua normal |
| tgl_update / user_update | – | now / karyawan atau admin | Setiap update di atas |
| status_asset (item rusak / normal saat ada rusak) | proses | **proses (tidak berubah)** | Konfirmasi ada rusak |
| ket_rusak, kondisi_awal | – | **tidak diubah** | – |

Draft penambahan dan draft alokasi **tidak** mengubah `asset_data`.

---

## 23. API Documentation

| ID | Method | Endpoint | Function | Auth |
|---|---|---|---|---|
| API-001 | GET | `/asset/penambahan-karyawan/options` | Opsi karyawan, kategori, asset | Admin |
| API-002 | GET | `/asset/penambahan-karyawan` | List penambahan | Admin |
| API-003 | POST | `/asset/penambahan-karyawan` | Buat penambahan | Admin |
| API-004 | GET | `/asset/penambahan-karyawan/{id}` | Detail penambahan | Admin |
| API-005 | GET | `/asset/pengajuan-karyawan` | List pengajuan | Admin |
| API-006 | GET | `/asset/pengajuan-karyawan/{id}` | Detail pengajuan | Admin |
| API-007 | GET | `/asset/pengajuan-karyawan/{id}/options` | Opsi alokasi | Admin |
| API-008 | POST | `/asset/pengajuan-karyawan/{id}/tolak` | Tolak pengajuan | Admin |
| API-009 | POST | `/asset/pengajuan-karyawan/{id}/alokasi` | Alokasi / setujui | Admin |
| API-010 | GET | `/asset/karyawan/pengajuan/options` | Opsi kategori (legacy, tidak dipakai mobile) | JWT |
| API-011 | GET | `/asset/karyawan/pengajuan` | List pengajuan saya | JWT |
| API-012 | POST | `/asset/karyawan/pengajuan` | Buat pengajuan | JWT |
| API-013 | GET | `/asset/karyawan/pengajuan/{id}` | Detail pengajuan saya | JWT, pemilik |
| API-014 | GET | `/asset/karyawan/penambahan/{id}` | Detail untuk konfirmasi (kedua sumber) | JWT, pemilik |
| API-015 | POST | `/asset/karyawan/penambahan/{id}/konfirmasi` | Konfirmasi Normal/Rusak | JWT, pemilik |
| API-016 | PUT | `/asset/penambahan-karyawan/{id}` | Ubah / kirim draft penambahan | Admin |
| API-017 | PUT | `/asset/karyawan/pengajuan/{id}` | Ubah / kirim draft pengajuan saya | JWT, pemilik |

Endpoint terkait: `GET /asset/karyawan/onboarding` (Asset Saya, bagian `penambahan[]`) dan `GET /asset/summary` (`pengajuan_perlu_proses`, `pengajuan_pending`).
Endpoint hapus draft, batal, dan tindak lanjut rusak: ⚠️ Belum ditemukan. Edit draft tersedia lewat `PUT` (API-016, API-017).

Format response sama dengan modul asset lain: `{ "code", "message", "data" }`; error validasi `{ "code": "VALIDATION_ERROR", "message", "errors" }`. HTTP: 200/201, 401, 403, 404, 422, 500.

---

## 24. Detail Setiap API

Header umum: `Authorization: Bearer <token>`, `Content-Type: application/json`, `Accept: application/json`, `locale`, `versi-app`.

### API-001 — Opsi Penambahan
```http
GET /asset/penambahan-karyawan/options
```
Response `data`: `{ "karyawan": [{id_karyawan, nik, nama_karyawan}], "kategori_asset": [{id_asset_kategori_m, parent_id, flag_parent, nama_kategori}], "assets": [{id_asset_data, kode_asset, nama_asset, id_asset_kategori_m, kategori_asset, merek, deskripsi, kondisi_awal}] }`. Message: "Opsi penambahan asset berhasil ditemukan." Semua karyawan aktif ditampilkan (tidak dikecualikan seperti onboarding).

### API-002 — List Penambahan
```http
GET /asset/penambahan-karyawan?tab=pending&limit=20&offset=0
```
| Param | Rule |
|---|---|
| tab | `pending` \| `dikonfirmasi` \| `selesai` ("Filter penambahan asset tidak valid.") |
| limit | 1–50 ("Limit maksimal adalah 50 data.") |
| offset | ≥ 0 |

Tab: `pending` = draft atau proses belum dikonfirmasi karyawan; `dikonfirmasi` = proses & `tgl_approval` terisi; `selesai` = ditolak/dikonfirmasi.
```json
{
  "code": "SUCCESS",
  "message": "Data penambahan asset karyawan berhasil ditemukan.",
  "data": {
    "filters": [
      {"value": "pending", "label": "Pending"},
      {"value": "dikonfirmasi", "label": "Dikonfirmasi"},
      {"value": "selesai", "label": "Selesai"}
    ],
    "requests": [
      {
        "id": "CMP001202609004",
        "no_trx": "KRY-03-1-202609004",
        "id_asset_kategori_m": "CMP001202601002",
        "nama_kategori": "Laptop",
        "keterangan": "",
        "ket_tolak": "",
        "id_karyawan": "KRY0002",
        "nik_karyawan": "EMP-002",
        "nama_karyawan": "Sari Contoh",
        "tgl_pengajuan": "2026-09-26",
        "tgl_serah_terima": "2026-09-29",
        "jam_serah_terima": "09:00:00",
        "keterangan_serah_terima": "Ambil di ruang IT",
        "status_approval": "proses",
        "status_label": "Menunggu Diterima",
        "tab": "pending",
        "jumlah_asset": 1,
        "jumlah_pending": 1,
        "jumlah_rusak": 0,
        "tgl_approval": ""
      }
    ],
    "pagination": {"limit": 20, "offset": 0, "next_offset": 1, "has_more": false}
  }
}
```
Tanpa parameter → `data` berupa array.

### API-003 — Buat Penambahan
```http
POST /asset/penambahan-karyawan
```
```json
{
  "id_karyawan": "KRY0002",
  "asset_ids": ["CMP00120260100005"],
  "tgl_serah_terima": "2026-09-29",
  "jam_serah_terima": "09:00",
  "keterangan_serah_terima": "Ambil di ruang IT",
  "keterangan": "",
  "urgensi": "",
  "is_draft": false
}
```
| Field | Rule | Message |
|---|---|---|
| id_karyawan | required\|string\|max:150 | Karyawan wajib dipilih. |
| asset_ids | required\|array\|min:1 | Minimal satu asset wajib dipilih. |
| asset_ids.* | required\|string\|max:50\|distinct | Asset yang sama tidak boleh dipilih lebih dari sekali. |
| tgl_serah_terima | required (kirim) / nullable (draft), date_format Y-m-d | Tanggal serah terima wajib diisi. / Format tanggal serah terima harus YYYY-MM-DD. |
| jam_serah_terima | required (kirim) / nullable (draft), date_format H:i | Jam serah terima wajib diisi. / Format jam serah terima harus HH:MM. |
| keterangan_serah_terima | nullable\|string\|max:255 | – |
| keterangan | nullable\|string | – |
| urgensi | nullable\|string\|max:50 | – (legacy: mobile masih mengirim `''`; tidak ditampilkan) |
| is_draft | required\|boolean | Status draft wajib dikirim. |

Database impact:
```text
INSERT asset_trx_request_karyawan (sumber_pengajuan = admin)
INSERT asset_trx_request_karyawan_item
UPDATE asset_data (status_asset = proses)   -- hanya bila bukan draft
INSERT notif_data                            -- hanya bila bukan draft
```
```json
{ "code": "CREATED", "message": "Penambahan asset berhasil dikirim ke karyawan.",
  "data": { "id": "CMP001202609004", "no_trx": "KRY-03-1-202609004",
            "status_approval": "proses", "status_label": "Menunggu Diterima", "tab": "pending" } }
```
Draft: "Draft penambahan asset berhasil disimpan." Error 422: "Satu atau beberapa asset tidak tersedia untuk penambahan.", "…sedang diproses pada onboarding.", "…sedang diproses pada penambahan lain.", "Karyawan tidak ditemukan atau tidak aktif.", "Pengguna tidak terdaftar sebagai admin asset.", "Nomor transaksi untuk periode ini sudah mencapai batas." 500: "Terjadi kesalahan saat menyimpan penambahan asset."

### API-004 — Detail Penambahan (Admin)
```http
GET /asset/penambahan-karyawan/{id}
```
```json
{
  "code": "SUCCESS",
  "message": "Detail penambahan asset berhasil ditemukan.",
  "data": {
    "id": "CMP001202609004",
    "no_trx": "KRY-03-1-202609004",
    "id_asset_kategori_m": "CMP001202601002",
    "nama_kategori": "Laptop",
    "keterangan": "",
    "ket_tolak": "",
    "tgl_pengajuan": "2026-09-26",
    "tgl_serah_terima": "2026-09-29",
    "jam_serah_terima": "09:00:00",
    "keterangan_serah_terima": "Ambil di ruang IT",
    "status_approval": "proses",
    "tgl_approval": "",
    "status_label": "Menunggu Diterima",
    "tab": "pending",
    "jumlah_asset": 1,
    "karyawan": { "nik": "EMP-002", "nama_karyawan": "Sari Contoh", "jabatan": "Staff Finance", "cabang": "Kantor Pusat", "divisi": "Finance" },
    "items": [
      {
        "id_item": "CMP001202609010",
        "parent_id": "",
        "id_asset_data": "CMP00120260100005",
        "kode_asset": "LPT-0005",
        "nama_asset": "Laptop 14 inch",
        "deskripsi": "Charger, tas",
        "kategori_asset": "Laptop",
        "merek": "MerekX",
        "status_item": "belum_dicek",
        "ket_rusak": "",
        "perlu_dicek": true,
        "diganti": false
      }
    ]
  }
}
```
404: "Penambahan asset tidak ditemukan." (hanya sumber admin). `karyawan.jabatan` (v1.4) berasal dari join `master_jabatan`, `''` bila kosong; key `divisi` dipertahankan (label UI: Departemen). Response tidak memuat `urgensi`.

### API-005 — List Pengajuan (Admin)
```http
GET /asset/pengajuan-karyawan?tab=pengajuan&limit=20&offset=0
```
`tab`: `pengajuan` (belum disetujui, termasuk Draft Alokasi), `proses` (disetujui, ada item pending), `dikonfirmasi` (disetujui, tanpa item pending — kasus rusak), `selesai` (ditolak/dikonfirmasi). Draft karyawan tidak pernah tampil. Struktur `data` sama dengan API-002 dengan `filters` Pengajuan/Proses/Dikonfirmasi/Selesai. Message: "Data pengajuan asset karyawan berhasil ditemukan."

### API-006 — Detail Pengajuan (Admin)
```http
GET /asset/pengajuan-karyawan/{id}
```
Struktur sama dengan API-004 (`keterangan` = alasan; `nama_kategori` dan `nama_asset` = teks jenis asset, `id_asset_kategori_m` = `''` untuk data baru; tanpa `urgensi`), label/tab dari §14.2. 404 "Pengajuan asset tidak ditemukan." bila tidak ada atau masih draft.

### API-007 — Opsi Alokasi
```http
GET /asset/pengajuan-karyawan/{id}/options
```
Sama dengan API-001: `assets` memuat seluruh stok tersedia di gudang, **tanpa filter kategori** (v1.4). 404 "Opsi pengajuan asset tidak ditemukan." bila pengajuan bukan `proses` atau sudah disetujui. Message: "Opsi persetujuan pengajuan asset berhasil ditemukan."

### API-008 — Tolak Pengajuan
```http
POST /asset/pengajuan-karyawan/{id}/tolak
```
```json
{ "ket_tolak": "Stok laptop belum tersedia bulan ini" }
```
| Field | Rule | Message |
|---|---|---|
| ket_tolak | required\|string\|max:255 | Alasan penolakan wajib diisi. |

Database impact: `DELETE item` (bila ada draft alokasi), `UPDATE header` (ditolak, ket_tolak, user_approval, tgl_approval), `INSERT notif_data`.
```json
{ "code": "UPDATED", "message": "Pengajuan asset berhasil ditolak.",
  "data": { "id": "CMP001202609005", "no_trx": "KRY-03-1-202609005",
            "status_approval": "ditolak", "status_label": "Ditolak", "tab": "selesai" } }
```
422: "Pengajuan asset tidak ditemukan.", "Pengajuan asset sudah tidak menunggu persetujuan." 500: "Terjadi kesalahan saat menolak pengajuan asset."

### API-009 — Alokasi / Setujui Pengajuan
```http
POST /asset/pengajuan-karyawan/{id}/alokasi
```
```json
{
  "asset_ids": ["CMP00120260100007"],
  "tgl_serah_terima": "2026-09-30",
  "jam_serah_terima": "10:00",
  "keterangan_serah_terima": "Serah terima di ruang IT lantai 2",
  "is_draft": false
}
```
Alokasi **tidak dibatasi kategori** (v1.4): asset apa pun dari stok boleh dipilih. Validasi sama dengan API-003 untuk `asset_ids`, jadwal, `keterangan_serah_terima`, `is_draft` (message: "Data persetujuan pengajuan belum lengkap.").

Database impact:
```text
DELETE asset_trx_request_karyawan_item (item lama)
INSERT asset_trx_request_karyawan_item
UPDATE asset_trx_request_karyawan (jadwal, ket_tolak='', user_approval/tgl_approval bila kirim)
UPDATE asset_data (status_asset = proses)   -- bila bukan draft
INSERT notif_data                            -- bila bukan draft
```
```json
{ "code": "SUCCESS", "message": "Pengajuan asset berhasil disetujui.",
  "data": { "id": "CMP001202609005", "no_trx": "KRY-03-1-202609005", "status_approval": "proses",
            "status_label": "Menunggu Konfirmasi", "tab": "proses", "jumlah_asset": 1 } }
```
Draft: code `UPDATED`, "Draft alokasi pengajuan asset berhasil disimpan.", label "Draft Alokasi", tab `pengajuan`.
422: "Pengajuan asset sudah tidak dapat dialokasikan.", "Satu atau beberapa asset tidak tersedia.", "…sedang diproses pada onboarding.", "…sedang diproses pada penambahan lain.", "Tanggal/Jam serah terima wajib diisi." 500: "Terjadi kesalahan saat menyetujui pengajuan asset."

### API-010 — Opsi Pengajuan (Karyawan) — legacy
```http
GET /asset/karyawan/pengajuan/options
```
```json
{
  "code": "SUCCESS",
  "message": "Opsi pengajuan asset berhasil ditemukan.",
  "data": {
    "kategori_asset": [
      {"id_asset_kategori_m": "CMP001202601001", "parent_id": "", "flag_parent": 1, "nama_kategori": "Elektronik"},
      {"id_asset_kategori_m": "CMP001202601002", "parent_id": "CMP001202601001", "flag_parent": 0, "nama_kategori": "Laptop"}
    ]
  }
}
```

**Catatan v1.4:** `urgensi` dihapus dari response. `kategori_asset` dipertahankan hanya untuk klien lama; mobile v1.4 tidak lagi memanggil endpoint ini.

### API-011 — List Pengajuan Saya
```http
GET /asset/karyawan/pengajuan?tab=pengajuan&limit=20&offset=0
```
`tab`: `pengajuan` (draft + menunggu admin), `dikonfirmasi` (disetujui, belum selesai), `ditolak`, `selesai`. Sebelum disetujui: jadwal dikosongkan dan `jumlah_asset = 0`. Filters: Pengajuan/Dikonfirmasi/Ditolak/Selesai.

### API-012 — Buat Pengajuan
```http
POST /asset/karyawan/pengajuan
```
```json
{ "jenis_asset": "Laptop", "keterangan": "Laptop lama sering mati, butuh untuk presentasi klien",
  "is_draft": false }
```
| Field | Rule | Message |
|---|---|---|
| jenis_asset | required (kirim) / nullable (draft), string, trim, max:150 | Jenis asset wajib diisi. |
| keterangan | required (kirim) / nullable (draft) | Alasan pengajuan wajib diisi. |
| id_asset_kategori_m, urgensi | – (diabaikan) | Field dari klien lama tidak divalidasi dan tidak disimpan. |
| is_draft | required\|boolean | Status draft wajib dikirim. |

Database impact: `INSERT asset_trx_request_karyawan` (sumber karyawan, tanpa item; `nama_kategori = jenis_asset`, `id_asset_kategori_m = NULL`, `urgensi = NULL`); `INSERT notif_data` (bukan draft).
```json
{ "code": "CREATED", "message": "Pengajuan asset berhasil dikirim ke admin.",
  "data": { "id": "CMP001202609005", "no_trx": "KRY-03-1-202609005",
            "status_approval": "proses", "status_label": "Menunggu Konfirmasi", "tab": "pengajuan" } }
```
422: "Karyawan tidak ditemukan atau tidak aktif." Kegagalan validasi: 422 `VALIDATION_ERROR` "Jenis asset wajib diisi." / "Alasan pengajuan wajib diisi." (bila lebih dari satu field gagal, message induk "Data pengajuan asset belum lengkap."). 500: "Terjadi kesalahan saat menyimpan pengajuan asset."

**Kompatibilitas klien lama (< v1.4):** submit tanpa `jenis_asset` ditolak 422 "Jenis asset wajib diisi."; `id_asset_kategori_m` dan `urgensi` dari klien lama diabaikan. Draft lama mempertahankan `nama_kategori` yang tersimpan (lihat API-017). Response list/detail mengisi `nama_kategori` dan `nama_asset` dengan teks jenis asset; `id_asset_kategori_m` bernilai `''` untuk data baru.

### API-013 — Detail Pengajuan Saya
```http
GET /asset/karyawan/pengajuan/{id}
```
Struktur seperti API-004 + `isu_rusak`; sebelum disetujui `items = []`, jadwal kosong. 404 "Pengajuan asset tidak ditemukan."

### API-014 — Detail untuk Konfirmasi
```http
GET /asset/karyawan/penambahan/{id}
```
Berlaku untuk kedua sumber: transaksi milik karyawan, bukan draft; pengajuan harus sudah disetujui. Struktur seperti API-004. 404 "Penambahan asset tidak ditemukan."

### API-015 — Konfirmasi Asset
```http
POST /asset/karyawan/penambahan/{id}/konfirmasi
```
```json
{ "items": [ { "id_item": "CMP001202609010", "status_item": "normal" } ] }
```
| Field | Rule | Message |
|---|---|---|
| items | required\|array\|min:1 | Kondisi asset wajib diisi. |
| items.*.id_item | required\|string\|max:50\|distinct | Asset yang sama tidak boleh dikirim lebih dari sekali. |
| items.*.status_item | required\|in:normal,rusak | Kondisi setiap asset wajib dipilih. / Kondisi asset harus normal atau rusak. |
| items.*.ket_rusak | nullable\|max:255\|required_if rusak | Alasan kerusakan wajib diisi. |

Database impact:
```text
UPDATE asset_trx_request_karyawan_item (status_item, ket_rusak)
UPDATE asset_trx_request_karyawan (status_approval, user_approval, tgl_approval)
UPDATE asset_data (terpakai, pemegang karyawan)   -- hanya jika semua normal
INSERT asset_trx_log (request_karyawan)                -- hanya jika semua normal
INSERT notif_data (semua admin)
```
```json
{ "code": "UPDATED", "message": "Penambahan asset berhasil dikonfirmasi.",
  "data": { "id": "CMP001202609004", "no_trx": "KRY-03-1-202609004",
            "status_approval": "dikonfirmasi", "status_label": "Selesai", "tab": "selesai" } }
```
Ada rusak: "Laporan kerusakan penambahan asset berhasil dikirim.", `status_approval: proses`, label "Isu Rusak", tab `dikonfirmasi`.
422: "Penambahan asset tidak ditemukan.", "Penambahan asset sudah tidak menunggu konfirmasi.", "Pengajuan asset belum disubmit oleh admin.", "Karyawan tidak ditemukan atau tidak aktif.", "Tidak ada asset penambahan yang perlu dicek.", "Seluruh asset penambahan wajib dicek.", "Alasan kerusakan wajib diisi.", "Satu atau beberapa asset penambahan tidak ditemukan." 500: "Terjadi kesalahan saat mengonfirmasi penambahan asset."

---

### API-016 — Ubah / Kirim Draft Penambahan
```http
PUT /asset/penambahan-karyawan/{id}
```
Body dan validasi sama dengan API-003 (`id_karyawan`, `asset_ids[]`, jadwal serah terima, `is_draft`). Hanya untuk penambahan sumber **admin** berstatus `draft`; selain itu 422 "Hanya draft penambahan langsung yang dapat diperbarui.". `id`, `no_trx`, dan tanggal pengajuan dipertahankan; item lama dihapus lalu ditulis ulang. Dengan `is_draft=false` status menjadi `proses`, asset dikunci (`asset_data.status_asset = proses`), dan notifikasi `penambahan.dikirim` dikirim.

Response: code `UPDATED`, "Draft penambahan asset berhasil disimpan." (draft) atau "Penambahan asset berhasil dikirim ke karyawan.".

### API-017 — Ubah / Kirim Draft Pengajuan Saya
```http
PUT /asset/karyawan/pengajuan/{id}
```
Body dan validasi sama dengan API-012 (`jenis_asset`, `keterangan`, `is_draft`). Pada PUT draft (`is_draft=true`) tanpa key `jenis_asset`, `nama_kategori` yang tersimpan dipertahankan (draft lama tetap utuh); bila key dikirim, nilainya menggantikan `nama_kategori` dan `id_asset_kategori_m` di-NULL-kan, `urgensi` selalu NULL. Saat `is_draft=false`, `jenis_asset` dan `keterangan` wajib (422 "Jenis asset wajib diisi." / "Alasan pengajuan wajib diisi."; message induk "Data pengajuan asset belum lengkap."). Hanya untuk pengajuan milik pemanggil yang masih `draft`, belum disetujui (`tgl_approval` kosong), dan belum punya item; selain itu 422 "Pengajuan asset sudah tidak dapat diubah karena bukan draft." dan 404 "Pengajuan asset tidak ditemukan." bila bukan miliknya. Dengan `is_draft=false` status menjadi `proses`, `tgl_pengajuan` diisi hari ini, dan notifikasi `pengajuan.diajukan` dikirim.

Response: code `UPDATED`, "Draft pengajuan asset berhasil disimpan." atau "Pengajuan asset berhasil dikirim ke admin.".

## 25. Mobile Architecture

```text
Page → Bloc → Api Service → ApiClient (Dio + Bearer JWT) → Backend
```

| Layer | File | Function |
|---|---|---|
| Api | `Repository/Service/assetRequestKaryawanApi.dart` | Penambahan admin: getOptions, getRequests, store, getDetail |
| Api | `Repository/Service/assetPengajuanAdminApi.dart` | Pengajuan admin: getRequests, getDetail, getOptions, reject, allocate |
| Api | `Repository/Service/assetPengajuanKaryawanApi.dart` | Pengajuan karyawan: getRequests, store, update (draft), getDetail (getOptions dihapus v1.4) |
| Api | `Repository/Service/assetSayaApi.dart` | getPenambahanDetail, konfirmasiPenambahan |
| Model | `Repository/Model/Asset/mAssetRequestKaryawan.dart` | AssetRequestApproval, AssetRequestKaryawanRecord/Item/Employee (field `jabatan`)/Page, AssetRequestKaryawanRequest, AssetKonfirmasiPenambahanRequest |
| Model | `Repository/Model/Asset/mAssetPengajuanKaryawan.dart` | AssetPengajuanKaryawanRequest (`jenisAsset`, `keterangan`, `isDraft` → `jenis_asset`, `keterangan`, `is_draft`); model Options dan Urgensi dihapus v1.4 |
| Model | `Repository/Model/Asset/mAssetPengajuanAdmin.dart` | TolakRequest, AlokasiRequest |
| Bloc | `AssetRequestKaryawanBloc` | List/paging, options, detail, submit penambahan |
| Bloc | `AssetPengajuanAdminBloc` | List/paging, detail, options, reject, allocate |
| Bloc | `AssetPengajuanKaryawanBloc` | List/paging, draft, detail, submit pengajuan (tanpa options sejak v1.4) |
| Bloc | `AssetKonfirmasiPenambahanBloc` | Load detail, submit konfirmasi |
| UI | `UI/Modul/Asset/AssetAdmin/AssetAdminKaryawan/Penambahan/*` | List, form, detail penambahan |
| UI | `UI/Modul/Asset/AssetAdmin/AssetAdminKaryawan/Pengajuan/*` | List, detail, form setujui |
| UI | `UI/Modul/Asset/AssetSaya/Pengajuan/*` | List, form, detail pengajuan karyawan |
| UI | `UI/Modul/Asset/AssetSaya/Onboarding/HalamanKonfirmasiPenambahanAsset.dart` | Konfirmasi (parameter `isPengajuan`) |

Routes:
| Route | Page | Event awal |
|---|---|---|
| `/asset_admin_penambahan_karyawan` | List Penambahan | LoadAssetRequestKaryawanList (tab pending) |
| `/asset_admin_add_penambahan_karyawan` | Form Penambahan | LoadAssetRequestKaryawanOptions |
| `/asset_admin_detail_penambahan_karyawan` | Detail Penambahan | LoadAssetRequestKaryawanDetail(id) |
| `/asset_admin_pengajuan_karyawan` | List Pengajuan admin | LoadAssetPengajuanAdminList (tab pengajuan) |
| `/asset_admin_detail_pengajuan_karyawan` | Detail Pengajuan admin (title "Konfirmasi Pengajuan Asset") | LoadAssetPengajuanAdminDetail(id) |
| `/asset_admin_setujui_pengajuan_karyawan` | Form Setujui | LoadAssetPengajuanAdminOptions(id), argumen record |
| `/asset_pengajuan` | List Pengajuan karyawan | LoadAssetPengajuanKaryawanList |
| `/asset_tambah_pengajuan` | Form Pengajuan | Form baru: tanpa event awal; `LoadAssetPengajuanKaryawanDraft(id)` saat ubah draft |
| `/asset_detail_pengajuan` | Detail Pengajuan karyawan | LoadAssetPengajuanKaryawanDetail(id) |
| `/asset_konfirmasi_pengajuan` | Konfirmasi (isPengajuan: true) | LoadAssetKonfirmasiPenambahan(id) |
| `/asset_konfirmasi_penambahan` | Konfirmasi Penambahan | LoadAssetKonfirmasiPenambahan(id) |

---

## 26. Mobile Screen

### 26.1 Penambahan Asset (Admin) — list
Tab Pending / Dikonfirmasi / Selesai; kartu: no transaksi, "Tanggal Penambahan", badge status, `nik - nama`, "{jumlah} Asset", "Jadwal Serah Terima" (bila ada). Tombol + ("Tambah penambahan asset"). Tap → detail.

### 26.2 Tambah Penambahan Asset (Admin)
| Field UI | Widget | Required | Source |
|---|---|:---:|---|
| Pilih Karyawan * | Field read-only → bottom sheet | Ya (draft & kirim) | `options.karyawan` |
| Penambahan Asset → "Tambah Asset" | Bottom sheet pohon kategori → asset | Ya, min 1 | `options.kategori_asset`, `options.assets` |
| Jadwal Serah Terima | Date picker (mulai hari ini) | Ya saat kirim | Input |
| Jam Serah Terima | Time picker (default 08:00) | Ya saat kirim | Input |
| Catatan Tambahan | Text field | Tidak | → `keterangan_serah_terima` |
| Tombol | Simpan Draft / Submit ke Karyawan | – | – |

### 26.3 Detail Penambahan Asset (Admin)
Tanggal Penambahan, badge status, Nomor Pengajuan; Data Karyawan (nama, NIK, Jabatan, Cabang, Departemen); Jadwal Serah Terima (tanggal, jam, catatan); daftar asset. **Tidak ada tombol aksi** dan **kondisi item (Normal/Rusak) tidak ditampilkan**.

**Alur status**: `AssetAlurStepper.pengajuan` (Pengajuan → Alokasi → Konfirmasi → Selesai; berpindah dari Alokasi ke Konfirmasi setelah admin mengalokasikan asset; disembunyikan bila ditolak). File: `Shared/Widgets/AssetAlurWidgets.dart`.

### 26.4 Pengajuan Asset (Admin) — list
Tab Pengajuan / Proses / Dikonfirmasi / Selesai; kartu menampilkan tanggal, status, karyawan, jadwal (bila ada); tanpa Urgensi (dihapus v1.4).

### 26.5 Konfirmasi Pengajuan Asset (Admin, detail)
Tanggal Pengajuan & status; Data Karyawan (Jabatan, Cabang, Departemen); Pengajuan Asset (Jenis Asset sebagai teks, Alasan); Alasan Penolakan (bila ditolak); Jadwal Serah Terima (bila ada); daftar asset (bila dialokasikan). Tombol **Tolak** (merah; bottom sheet "Alasan Penolakan" wajib → Kirim) dan **Setujui** (hijau), keduanya berbentuk pil tinggi 44, tampil jika status `proses` dan tab `pengajuan` (termasuk Draft Alokasi).

Stepper alur status sama seperti §26.3 (`AssetAlurStepper.pengajuan`).

### 26.6 Setujui Pengajuan Asset (Admin)
Ringkasan Jenis Asset (teks) dan alasan, tanpa Urgensi; "Penambahan Asset" + tombol "Tambah Asset" (lebar penuh, pil hijau; picker memakai **seluruh kategori** tanpa filter jenis pengajuan, draft alokasi terisi otomatis); Jadwal Serah Terima & Jam (wajib saat kirim; **kosong sampai admin mengisi**: nilai `0000-00-00` / `00:00:00` dari DB dianggap kosong, tanggal bertahun < 2000 dan jamnya dikosongkan, sehingga tidak lagi tampil `30-11-0001` / `00:00`); Catatan Tambahan; tombol **Simpan Draft** (biru terisi) / **Submit ke PIC** (outline biru), keduanya pil tinggi 44.

### 26.7 Pengajuan Asset (Karyawan) — list
Tab Pengajuan / Dikonfirmasi / Ditolak / Selesai; kartu: tanggal, status (Draft, Menunggu Konfirmasi, Dikonfirmasi Admin, Butuh Proses, Ditolak, Selesai), alasan (tanpa Urgensi). Tap: bila disetujui & ada item pending → Konfirmasi Pengajuan Asset, selain itu → Detail.

### 26.8 Tambah Pengajuan Asset (Karyawan)
| Field UI | Widget | Required | Source |
|---|---|:---:|---|
| Jenis Asset | TextFormField teks bebas (maxLength 150, counter disembunyikan; error server key `jenis_asset`) | Ya saat kirim (draft boleh kosong) | Input |
| Alasan | Text field multiline | Ya saat kirim | Input |
| Urgensi | Field dihapus (v1.4) | – | – |
| Tombol | Simpan Draft / Submit ke Admin | – | – |

### 26.9 Detail Pengajuan Asset (Karyawan)
Nomor Pengajuan, data karyawan (Jabatan, Cabang, Departemen), Jenis Asset, Alasan, Alasan Penolakan (bila ditolak), Jadwal, asset beserta kondisi Normal/Rusak dan Report Kerusakan.

Stepper alur status sama seperti §26.3 (`AssetAlurStepper.pengajuan`).

### 26.10 Konfirmasi Penambahan / Pengajuan Asset (Karyawan)
Info transaksi (Tanggal & Kode Penambahan / Nomor Pengajuan), Data Karyawan (Jabatan, Cabang, Departemen), Jadwal Serah Terima, (Pengajuan: Jenis Asset, Alasan), "Penting! Periksa kondisi asset…", per asset Kelengkapan + tombol **Normal/Rusak** + "Report Kerusakan *" (maks 255), pernyataan tanggung jawab, tombol **Konfirmasi Asset**.

### 26.11 Asset Saya
Menu **Pengajuan**; banner **Penambahan Asset** untuk transaksi pertama yang menunggu dicek.

---

## 27. Mobile → API

| UI Field | Mobile Variable | JSON Field | Type | Required |
|---|---|---|---|:---:|
| Pilih Karyawan (penambahan) | `_selectedEmployee.idKaryawan` | `id_karyawan` | string | Ya |
| Tambah Asset (penambahan/alokasi) | `_selectedAssets[].idAssetData` | `asset_ids[]` | array | Ya |
| Jadwal Serah Terima | `tanggalPeminjamanApi(_handoverDate)` | `tgl_serah_terima` | Y-m-d | Ya saat kirim |
| Jam Serah Terima | `_timeValue(_handoverTime)` | `jam_serah_terima` | H:i | Ya saat kirim |
| Catatan Tambahan (penambahan/alokasi) | `_notesController.text.trim()` | `keterangan_serah_terima` | string | Tidak |
| (tidak ada field) | konstanta `''` | `keterangan`, `urgensi` (penambahan) | string | – |
| Simpan Draft / Submit | `isDraft` | `is_draft` | bool | Ya |
| Jenis Asset (pengajuan) | `_categoryController.text.trim()` | `jenis_asset` | string (maks 150) | Ya saat kirim |
| Alasan (pengajuan) | `_reasonController.text.trim()` | `keterangan` | string | Ya saat kirim |
| Urgensi (pengajuan) | – (field dihapus v1.4) | – | – | – |
| Alasan Penolakan | `AssetPengajuanAdminTolakRequest.keterangan` (trim) | `ket_tolak` | string | Ya |
| Normal / Rusak | `statusByItem[idItem]` | `items[].status_item` | string | Ya |
| Report Kerusakan | `ketRusakByItem[idItem]` | `items[].ket_rusak` | string | Ya bila rusak |

---

## 28. API → Mobile

| API JSON | Mobile Model | UI |
|---|---|---|
| `requests[]` / array | `AssetRequestKaryawanPage.records` | Kartu list |
| `id`, `no_trx` | `id`, `noTrx` | Navigasi, "Nomor Pengajuan"/"Kode Penambahan" |
| `nama_kategori` (+ `id_asset_kategori_m` legacy, `''` untuk data baru) | `namaKategori`, `idAssetKategori` | "Jenis Asset" (teks bebas) |
| `keterangan` | `keterangan` | "Alasan" |
| `urgensi` (legacy) | Tidak dibaca untuk Pengajuan | Tidak ditampilkan (dihapus v1.4) |
| `ket_tolak` | `ketTolak` | "Alasan Penolakan" |
| `tgl_pengajuan` | `tanggalPengajuan` | "Tanggal Pengajuan/Penambahan" |
| `tgl_serah_terima`, `jam_serah_terima`, `keterangan_serah_terima` | `tanggalSerahTerima`, `jamSerahTerima`, `keteranganSerahTerima` (`hasSchedule` mengabaikan `0000-00-00`/`00:00:00`) | "Jadwal Serah Terima" |
| `status_approval` | `statusApproval` (`parseAssetRequestApproval`) | Badge, kondisi tombol |
| `status_label`, `tab` | `statusLabel`, `tab` | Badge (dipetakan ulang di mobile) |
| `jumlah_asset`, `jumlah_pending`, `jumlah_rusak` | idem | "{jumlah} Asset", kondisi konfirmasi, "Butuh Proses" |
| `tgl_approval` | `tanggalApproval` | Syarat konfirmasi pengajuan |
| `isu_rusak` | `isuRusak` (atau dihitung dari item) | "Butuh Proses" |
| `karyawan.*` (id_karyawan, nik, nama_karyawan, **jabatan**, cabang, divisi) / `nik_karyawan`, `nama_karyawan` | `employee` (`AssetRequestKaryawanEmployee`; `jabatan` dibaca dari `jabatan` atau `nama_jabatan`) | Data Karyawan: Jabatan, Cabang, Departemen |
| `items[]` | `AssetRequestKaryawanItem` (idItem, kodeAsset, namaAsset, kategoriAsset, merek, deskripsi, statusItem, ketRusak, perluDicek) | Daftar asset, kondisi |
| `filters[]` | `AssetListFilter` | Tab |
| `pagination` | `nextOffset`, `hasMore` | Infinite scroll |

---

## 29. Mapping Mobile → API → Database

| UI | Mobile Variable | JSON | Backend | Table | Column |
|---|---|---|---|---|---|
| Pilih Karyawan | `idKaryawan` | `id_karyawan` | `$karyawan` | header | id_karyawan, nik_karyawan, nama_karyawan |
| Tambah Asset | `assetIds` | `asset_ids[]` | `$assets` | item | id_asset_data (+ snapshot) |
| Tambah Asset | `assetIds` | `asset_ids[]` | `$assetIds` | asset_data | status_asset = proses (bila kirim) |
| Tambah Asset (kategori) | – | – | `$categoryIds` | header | id_asset_kategori_m, nama_kategori |
| Jadwal Serah Terima | `tanggalSerahTerima` | `tgl_serah_terima` | `$tanggalSerahTerima` | header | tgl_serah_terima |
| Jam Serah Terima | `jamSerahTerima` | `jam_serah_terima` | `Carbon H:i → H:i:s` | header | jam_serah_terima |
| Catatan Tambahan | `keteranganSerahTerima` | `keterangan_serah_terima` | trim | header | keterangan_serah_terima |
| Jenis Asset | `jenisAsset` | `jenis_asset` | `$jenisAsset` (trim) | header | nama_kategori (`id_asset_kategori_m` = NULL) |
| Alasan | `keterangan` | `keterangan` | trim | header | keterangan |
| Urgensi | – (dihapus) | – (diabaikan) | NULL | header | urgensi = NULL (legacy) |
| Simpan Draft / Submit | `isDraft` | `is_draft` | `$statusApproval` | header | status_approval |
| Alasan Penolakan | `keterangan` | `ket_tolak` | `$reason` | header | ket_tolak, status_approval = ditolak |
| Normal / Rusak | `statusByItem` | `items[].status_item` | `$statusByItem` | item | status_item |
| Report Kerusakan | `ketRusakByItem` | `items[].ket_rusak` | `$ketRusakByItem` | item | ket_rusak |
| (JWT admin) | – | Authorization | `$actor` | header/item | user_input, user_update, user_approval |
| (JWT karyawan konfirmasi) | – | Authorization | `$karyawan` | header / asset_data / asset_trx_log | user_approval, tgl_approval / id_pemegang_asset / id_pemegang |

---

## 30. Mapping Database → API → Mobile

| Table | Column | Backend | API JSON | Mobile | UI |
|---|---|---|---|---|---|
| header | no_trx | requestQuery / getDetail | `no_trx` | noTrx | Nomor Pengajuan / Kode Penambahan |
| header | nama_kategori | 〃 | `nama_kategori` | namaKategori | Jenis Asset |
| header | keterangan | 〃 | `keterangan` | keterangan | Alasan |
| header | urgensi (legacy) | – | tidak dikirim | tidak dibaca | tidak ditampilkan |
| header | ket_tolak | 〃 | `ket_tolak` | ketTolak | Alasan Penolakan |
| header | tgl_serah_terima, jam_serah_terima | 〃 | idem | tanggalSerahTerima, jamSerahTerima | Jadwal Serah Terima |
| header | status_approval + tgl_approval + item | *StatusMetadata | `status_approval`, `status_label`, `tab` | statusApproval, statusLabel, tab | Badge, tab, tombol |
| item | COUNT / SUM status | requestQuery | `jumlah_asset`, `jumlah_pending`, `jumlah_rusak` | idem | "{jumlah} Asset", Butuh Proses |
| item | nama_asset, kategori_asset, deskripsi_asset | getDetail | `nama_asset`, `kategori_asset`, `deskripsi` | namaAsset, kategoriAsset, deskripsi | Kartu asset, Kelengkapan |
| item | status_item, ket_rusak | getDetail | `status_item`, `perlu_dicek`, `ket_rusak` | statusItem, perluDicek, ketRusak | Normal/Rusak, Report Kerusakan |
| asset_data | kode_asset, merek | LEFT JOIN | `kode_asset`, `merek` | kodeAsset, merek | Subtitle asset |
| data_karyawan → master | jabatan, cabang, departemen | LEFT JOIN (`master_jabatan` lewat `data_karyawan.id_jabatan`, `master_cabang`, `master_departemen`) | `karyawan.jabatan`, `karyawan.cabang`, `karyawan.divisi` | employee.jabatan/cabang/divisi | Data Karyawan (Jabatan, Cabang, Departemen) |

---

## 31. Validation

### 31.1 Mobile
| Lokasi | Aturan | Pesan |
|---|---|---|
| Form Penambahan | Karyawan wajib (draft & kirim) | "Karyawan wajib dipilih" |
| Form Penambahan / Setujui | Min 1 asset (draft & kirim) | "Tambahkan minimal satu asset." |
| Form Penambahan / Setujui | Tanggal & jam wajib saat kirim; tanggal ≥ hari ini | "Tanggal serah terima wajib dipilih." / "Jam serah terima wajib dipilih." |
| Form Setujui | Picker memakai seluruh kategori (tanpa filter jenis pengajuan) | – |
| Form Pengajuan | Jenis asset (TextFormField, maks 150) wajib saat kirim; draft boleh kosong; error server key `jenis_asset` | "Jenis asset wajib diisi." |
| Form Pengajuan | Alasan wajib saat kirim | "Alasan pengajuan wajib diisi." |
| Tolak | Alasan wajib | "Alasan penolakan wajib diisi." |
| Konfirmasi | Semua item pending dinilai; rusak wajib report (≤255) | Tombol nonaktif sampai lengkap |
| Semua submit | Abaikan saat `submitting` | – |

### 31.2 Backend
Lihat §9 dan §24.

### 31.3 Database
ENUM `sumber_pengajuan`, `status_approval`, `status_item`; UNIQUE `no_trx` (DBML); NOT NULL default untuk jadwal (`0000-00-00`, `00:00:00`); `id_asset_kategori_m` dan `urgensi` NULL diperbolehkan setelah SQL v1.4 dijalankan.

### 31.4 Ringkasan per Field
| Field | Mobile | Backend | Database |
|---|---|---|---|
| id_karyawan | Wajib | required, karyawan aktif | varchar(150) |
| asset_ids | Min 1, picker exclude duplikat | required, distinct, tersedia+gudang, tidak aktif (alokasi tanpa filter kategori) | item.id_asset_data |
| tgl/jam_serah_terima | Wajib saat kirim, tanggal ≥ hari ini | required saat kirim, format | date/time NOT NULL |
| jenis_asset (pengajuan) | Teks bebas, maks 150, wajib saat kirim | required saat kirim, string, trim, max:150 | `nama_kategori` varchar(255) |
| keterangan (pengajuan) | Wajib saat kirim | required saat kirim | text |
| urgensi | – (dihapus) | – (diabaikan) | varchar(50) NULL (legacy) |
| ket_tolak | Wajib | required, ≤255 | varchar(255) |
| status_item / ket_rusak | Wajib / wajib bila rusak | in / required_if | enum / varchar(255) |

---

## 32. Error Handling

| Error | HTTP | Backend | Mobile |
|---|---:|---|---|
| Field tidak valid | 422 | VALIDATION_ERROR + errors | Error per field / snackbar warning |
| Aturan bisnis (asset tidak tersedia, status tidak valid, dll.) | 422 | BUSINESS_RULE_VIOLATION | Snackbar error |
| Detail / options tidak ditemukan | 404 | NOT_FOUND | Halaman error + Coba Lagi |
| Token tidak valid/expired | 401 | UNAUTHORIZED | Refresh token & retry otomatis |
| Bukan admin | 403 | FORBIDDEN | Pesan error |
| Exception | 500 | SYSTEM_ERROR + Log::error | Snackbar |
| Timeout / koneksi | – | – | "Tidak dapat terhubung ke server. Periksa koneksi Anda." |
| Double submit | – | tanpa idempotency | Bloc abaikan saat submitting |
| Notifikasi gagal | – | di-log, transaksi tetap sukses | – |

---

## 33. Audit Trail

| Data | Disimpan? | Lokasi |
|---|:---:|---|
| Pembuat & waktu | ✅ | `user_input`, `tgl_input` (admin atau karyawan) |
| Pengubah terakhir | ✅ | `user_update`, `tgl_update` |
| Penyetuju/penolak admin | ⚠️ sebagian | `user_approval`/`tgl_approval` — **ditimpa** karyawan saat konfirmasi |
| Alasan penolakan | ✅ | `ket_tolak` |
| Histori status | ❌ | ⚠️ Belum ditemukan |
| Draft alokasi sebelumnya | ❌ | Item dihapus & ditulis ulang |
| Asset log serah terima | ✅ | `asset_trx_log` (`request_karyawan`) hanya saat semua normal |
| Laporan rusak | ⚠️ | Hanya `status_item`/`ket_rusak` di item; tanpa `asset_trx_log`, tanpa perubahan `asset_data` |
| Error sistem | ✅ | `Log::error` |

---

## 34. Notification

| Event | Trigger | Penerima | Judul | Pesan | Rute mobile |
|---|---|---|---|---|---|
| `pengajuan.diajukan` | Karyawan submit pengajuan | Semua admin asset | Pengajuan Asset Baru | {nama} mengajukan asset {no_trx}. | `asset_admin_detail_pengajuan_karyawan` |
| `pengajuan.ditolak` | Admin tolak | Karyawan | Pengajuan Asset Ditolak | Pengajuan asset {no_trx} ditolak admin. | `asset_detail_pengajuan` |
| `pengajuan.disetujui` | Admin submit alokasi | Karyawan | Pengajuan Asset Disetujui | Pengajuan asset {no_trx} disetujui. Silakan cek dan konfirmasi asset. | `asset_konfirmasi_pengajuan` |
| `penambahan.dikirim` | Admin submit penambahan | Karyawan | Penambahan Asset Baru | Anda menerima penambahan asset {no_trx}. | `asset_konfirmasi_penambahan` |
| `penambahan.dikonfirmasi` | Konfirmasi semua normal | Semua admin | Asset Dikonfirmasi Karyawan | Asset {no_trx} telah dikonfirmasi oleh {nama}. | Sesuai sumber: `asset_admin_detail_penambahan_karyawan` / `asset_admin_detail_pengajuan_karyawan` |
| `penambahan.rusak` | Konfirmasi ada rusak | Semua admin | Laporan Asset Rusak | {nama} melaporkan asset rusak pada {no_trx}. | Sesuai sumber (sama seperti di atas) |

Mekanisme sama dengan modul asset lain: INSERT `notif_data` + push FCM, dipanggil setelah commit, kegagalan hanya di-log.

---

## 35. Security

| Aspek | Implementasi |
|---|---|
| Autentikasi | Bearer JWT (`jwt.api`), context dari token (`jwt.ctx`) |
| Isolasi company | Semua query `WHERE id_company = ctx.id_company` |
| Otorisasi admin | `accessError` + `findAssetAdmin` |
| Kepemilikan karyawan | Detail & konfirmasi memfilter `id_karyawan = ctx.id_karyawan` dan `sumber_pengajuan` |
| Visibilitas | Karyawan tidak melihat alokasi sebelum disetujui; admin tidak melihat draft karyawan |
| Concurrency | `lockCompany` + `lockForUpdate` |
| Input teks bebas | `jenis_asset` di-trim dan dibatasi 150 karakter; disimpan lewat Query Builder (parameter binding) dan ditampilkan sebagai teks biasa di mobile |
| Catatan | Validasi request dijalankan sebelum cek admin pada store/tolak/alokasi (non-admin dapat menerima 422, bukan 403) |

---

## 36. Manual Book User/Admin

> Label diambil dari `assets/id/asset.json`. ⚠️ Asumsi: halaman admin **Asset Karyawan** diakses dari menu **Asset Management**. Screenshot menyusul.

### A. Admin — Penambahan Asset
1. Buka **Asset Karyawan** → aksi **Penambahan** → halaman **Penambahan Asset** (tab Pending, Dikonfirmasi, Selesai).
2. Tekan **+** → **Tambah Penambahan Asset**.
3. **Pilih Karyawan**.
4. **Tambah Asset** → pilih kategori → pilih asset (boleh beberapa, boleh beda kategori).
5. Isi **Jadwal Serah Terima** dan **Jam Serah Terima**, serta **Catatan Tambahan** bila perlu.
6. Tekan **Submit ke Karyawan**. Asset langsung dikunci dan karyawan mendapat notifikasi.

> **Simpan Draft** menyimpan penambahan tanpa mengunci asset. Buka detail draft lalu tekan tombol ubah draft untuk melanjutkan dan mengirim; draft belum dapat dihapus dari aplikasi (lihat §41 RISK-003).

### B. Admin — Meninjau Pengajuan
1. Buka **Asset Karyawan** → aksi **Pengajuan** (atau panel **Pengajuan Asset Perlu Diproses** → **Tinjau Pengajuan**).
2. Tab **Pengajuan** → pilih pengajuan → halaman **Konfirmasi Pengajuan Asset** (Jenis Asset, Alasan).
3. **Menolak**: tekan **Tolak** → isi **Alasan Penolakan** → **Kirim**.
4. **Menyetujui**: tekan **Setujui** → halaman **Setujui Pengajuan Asset** → **Tambah Asset** (boleh asset apa pun dari stok; jenis yang diajukan hanya informasi) → isi jadwal (kosong sampai Anda mengisi) → **Submit ke PIC**.
   - **Simpan Draft** menyimpan pilihan sebagai **Draft Alokasi**; buka lagi dari tab Pengajuan (**Lanjutkan Alokasi**) untuk mengirim.
5. Pengajuan berpindah ke tab **Proses** (menunggu karyawan), lalu **Selesai** setelah karyawan mengonfirmasi semua asset Normal.

### C. Karyawan — Mengajukan Asset
1. Buka **Asset Saya** → menu **Pengajuan** → tekan **+**.
2. Ketik **Jenis Asset** (teks bebas, maks 150 karakter, mis. Laptop atau Monitor 24 inch) dan isi **Alasan**.
3. Tekan **Submit ke Admin**.
4. Pantau status di tab **Pengajuan**, **Dikonfirmasi**, **Ditolak**, atau **Selesai**. Jika ditolak, alasan tampil di detail.

### D. Karyawan — Menerima & Mengonfirmasi Asset
1. Buka notifikasi **Penambahan Asset Baru** / **Pengajuan Asset Disetujui**, banner **Penambahan Asset** di Asset Saya, atau pengajuan berstatus **Menunggu Konfirmasi** di tab Dikonfirmasi.
2. Periksa setiap asset secara fisik, pilih **Normal** atau **Rusak** (isi **Report Kerusakan** bila rusak).
3. Tekan **Konfirmasi Asset**.
   - Semua Normal → asset menjadi milik Anda, status **Selesai**.
   - Ada Rusak → status **Butuh Proses**; hubungi Admin Asset karena tindak lanjut belum tersedia di aplikasi.

---

## 37. Kondisi Button

| Button | Role | Halaman | Condition | Action |
|---|---|---|---|---|
| + Tambah penambahan asset | Admin | List Penambahan | Selalu | Form penambahan |
| Simpan Draft | Admin | Form Penambahan | Tidak submitting | POST `is_draft=true` |
| Submit ke Karyawan | Admin | Form Penambahan | Tidak submitting | POST `is_draft=false` |
| (tidak ada aksi) | Admin | Detail Penambahan | – | Read-only |
| Tolak | Admin | Detail Pengajuan | `status = proses` & tab `pengajuan` | Bottom sheet alasan → POST tolak (tombol merah, pil tinggi 44) |
| Setujui | Admin | Detail Pengajuan | Sama dengan Tolak | Form Setujui (tombol hijau, pil tinggi 44) |
| Simpan Draft | Admin | Form Setujui | Tidak submitting | POST alokasi `is_draft=true` (biru terisi, pil tinggi 44) |
| Submit ke PIC | Admin | Form Setujui | Tidak submitting | POST alokasi `is_draft=false` (outline biru, pil tinggi 44) |
| Tambah Asset | Admin | Form Setujui | Tidak submitting | Picker asset semua kategori (lebar penuh, pil hijau) |
| + Tambah Pengajuan Asset | Karyawan | List Pengajuan | Selalu | Form pengajuan |
| Simpan Draft / Submit ke Admin | Karyawan | Form Pengajuan | Tidak submitting | POST pengajuan |
| Tap kartu pengajuan | Karyawan | List Pengajuan | `proses` & `tgl_approval` & `jumlah_pending > 0` → Konfirmasi; selain itu → Detail | Navigasi |
| Normal / Rusak | Karyawan | Konfirmasi | Item `perlu_dicek`, tidak submitting | Set kondisi |
| Konfirmasi Asset | Karyawan | Konfirmasi | `status = proses` & ada item pending; aktif bila semua valid | POST konfirmasi |

---

## 38. Testing Scenario

> Hasil otomatis terakhir (2026-10-20): API `tests/Asset/AssetModelTest.php` 99 tes lulus (1293 assertion); mobile `test/asset/asset_pengajuan_karyawan_test.dart` dan `test/asset/asset_pengajuan_admin_test.dart` 38 tes lulus.

### Happy Path
| TC | Scenario | Precondition | Steps | Expected |
|---|---|---|---|---|
| TC-01 | Penambahan dikirim | Admin, karyawan aktif, asset tersedia | Isi form lengkap → Submit ke Karyawan | 201; `sumber=admin`, `proses`; item `belum_dicek`; asset `proses`; notif karyawan |
| TC-02 | Penambahan multi kategori | 2 asset beda kategori | Submit | `nama_kategori = Multi Kategori`, `id_asset_kategori_m = ''` |
| TC-03 | Konfirmasi penambahan normal | TC-01 | Karyawan Normal semua → Konfirmasi | `dikonfirmasi`; asset `terpakai`/karyawan; `asset_trx_log request_karyawan`; notif admin |
| TC-04 | Pengajuan dikirim | Karyawan aktif | Ketik jenis asset + alasan → Submit ke Admin | 201; `sumber=karyawan`; tanpa item; `nama_kategori` = teks jenis asset, `id_asset_kategori_m`/`urgensi` NULL; notif semua admin |
| TC-05 | Tolak pengajuan | TC-04 | Tolak + alasan | `ditolak`; `ket_tolak`; notif karyawan; tab Ditolak (karyawan) / Selesai (admin) |
| TC-06 | Draft alokasi | TC-04 | Setujui → pilih asset → Simpan Draft | Label "Draft Alokasi"; asset tetap `tersedia`; karyawan belum melihat asset |
| TC-07 | Submit alokasi | TC-04/06 | Submit ke PIC | `tgl_approval` terisi; asset `proses`; notif "Disetujui"; admin tab Proses |
| TC-08 | Konfirmasi pengajuan normal | TC-07 | Karyawan Normal → Konfirmasi | `dikonfirmasi`; asset diserahkan |

### Validation
| TC | Scenario | Expected |
|---|---|---|
| TC-09 | Penambahan tanpa karyawan | Mobile "Karyawan wajib dipilih"; API 422 |
| TC-10 | Penambahan tanpa asset | "Tambahkan minimal satu asset." / API 422 |
| TC-11 | Kirim tanpa jadwal | "Tanggal/Jam serah terima wajib…" |
| TC-12 | Format jam salah (`9.00`) | 422 "Format jam serah terima harus HH:MM." |
| TC-13 | Pengajuan tanpa jenis asset atau alasan | 422 "Jenis asset wajib diisi." / "Alasan pengajuan wajib diisi." |
| TC-14 | Klien lama mengirim `urgensi` dan `id_asset_kategori_m` | Diabaikan; tersimpan `nama_kategori = jenis_asset`, `id_asset_kategori_m = NULL`, `urgensi = NULL` |
| TC-15 | Submit tanpa `jenis_asset` (klien lama, API) | 422 "Jenis asset wajib diisi." (`test_submit_without_jenis_asset_is_rejected`) |
| TC-16 | Tolak tanpa alasan | 422 "Alasan penolakan wajib diisi." |
| TC-17 | Alokasi asset lintas kategori dari jenis yang diajukan (API) | Diterima; alokasi tidak dibatasi kategori (`test_admin_can_allocate_any_stock_asset_regardless_of_free_text_jenis`) |
| TC-18 | Konfirmasi sebagian item | 422 "Seluruh asset penambahan wajib dicek." |
| TC-19 | Rusak tanpa alasan | 422 "Alasan kerusakan wajib diisi." |

### Status
| TC | Scenario | Expected |
|---|---|---|
| TC-20 | Tolak pengajuan yang sudah disetujui | 422 "…sudah tidak menunggu persetujuan." |
| TC-21 | Alokasi ulang setelah disetujui | 422 "…sudah tidak dapat dialokasikan." |
| TC-22 | Options alokasi setelah disetujui | 404 |
| TC-23 | Konfirmasi pengajuan belum disetujui (API) | 422 "Pengajuan asset belum disubmit oleh admin." |
| TC-24 | Konfirmasi transaksi selesai | 422 "…sudah tidak menunggu konfirmasi." |
| TC-25 | Konfirmasi ada rusak | `proses`, karyawan "Butuh Proses", admin "Dikonfirmasi"; **tidak ada asset berubah** |
| TC-26 | Konfirmasi ulang setelah rusak | 422 "Tidak ada asset penambahan yang perlu dicek." |
| TC-27 | Admin buka detail draft pengajuan karyawan | 404 |

### Asset
| TC | Scenario | Expected |
|---|---|---|
| TC-28 | Asset sedang di onboarding | 422 "…sedang diproses pada onboarding." |
| TC-29 | Asset sedang di penambahan lain | 422 "…sedang diproses pada penambahan lain." |
| TC-30 | Asset pernah diserahkan via penambahan (header `dikonfirmasi`), lalu diretur ke gudang | 201, penambahan baru tersimpan (diperbaiki v1.1; test `test_returned_asset_from_confirmed_addition_can_be_added_again`) |
| TC-30a | Asset kembali `tersedia` tetapi header penambahan lama masih `proses` | 422 "…sedang diproses pada penambahan lain." |
| TC-31 | Asset di draft penambahan dipakai transaksi lain | Diizinkan (draft tidak mengunci) |

### API / Security / Network
| TC | Scenario | Expected |
|---|---|---|
| TC-32 | Tanpa token | 401 |
| TC-33 | Non-admin akses endpoint admin | 403 (atau 422 bila payload invalid) |
| TC-34 | Karyawan buka pengajuan orang lain | 404 |
| TC-35 | `limit=100` | 422 |
| TC-36 | Double tap Submit | Hanya satu request |
| TC-37 | Timeout setelah commit, submit ulang penambahan | 422 asset tidak tersedia (sudah `proses`) |
| TC-38 | Notifikasi dibuka | Rute sesuai §34 |
| TC-39 | PUT draft pengajuan tanpa key `jenis_asset` (draft lama) | `nama_kategori` tersimpan dipertahankan (`test_draft_update_without_jenis_asset_key_keeps_stored_jenis_asset`) |
| TC-40 | Baris lama berisi `id_asset_kategori_m` + `urgensi` dibaca | Terbaca normal; response tanpa `urgensi` (`test_legacy_request_row_with_category_and_urgensi_stays_readable`) |
| TC-41 | Buka layar Setujui saat DB mengirim `0000-00-00` / `00:00:00` | Field Jadwal dan Jam kosong sampai admin mengisi (bukan `30-11-0001` / `00:00`) |

---

## 39. Temuan Ketidaksesuaian

| ID | Component | Finding | Impact | Recommendation |
|---|---|---|---|---|
| INC-01 | Mobile ↔ DB | Form Penambahan: "Catatan Tambahan" dikirim sebagai `keterangan_serah_terima`; `keterangan` dan `urgensi` selalu `''` | Kolom `keterangan`/`urgensi` tidak dipakai untuk Penambahan | Dokumentasikan atau tambah field |
| INC-02 | Mobile | Detail Penambahan & Pengajuan admin **tidak menampilkan** `status_item`/`ket_rusak` | Admin tidak tahu asset mana yang rusak | Tampilkan kondisi item |
| INC-03 | Backend vs Onboarding | Request: all-or-nothing, tanpa penggantian, asset rusak tidak diubah jadi `rusak`. Onboarding: parsial, ada penggantian, asset rusak → `rusak` | Perilaku tidak konsisten antar fitur sejenis | Samakan aturan bisnis |
| INC-04 | Label | Admin "Dikonfirmasi" (tab dikonfirmasi) berarti karyawan melaporkan rusak; karyawan melihat "Butuh Proses"; response konfirmasi memberi label "Isu Rusak" | Tiga label berbeda untuk kondisi sama | Seragamkan label |
| INC-05 | API naming | Konfirmasi Pengajuan memakai endpoint `/karyawan/penambahan/{id}/konfirmasi` | Membingungkan | Dokumentasikan / alias |
| INC-06 | Mobile | Asset Saya hanya menampilkan banner penambahan **pertama** (`penambahans.first`) | Penambahan kedua dari admin hanya bisa diakses via notifikasi | Tampilkan semua |
| INC-07 | DB/Backend | Kolom item `parent_id` dan field API `diganti` selalu kosong/false | Dead field | Hapus atau implementasi penggantian |
| INC-08 | Backend | Enum `RequestKaryawanStatus` ada, tetapi model memakai konstanta sendiri (`STATUS_DISETUJUI = 'dikonfirmasi'`) | Dua sumber kebenaran | Pakai enum |
| INC-09 | Label | "Menunggu Konfirmasi" (karyawan) dipakai untuk menunggu admin **dan** menunggu karyawan mengecek | Ambigu | Bedakan label |
| INC-10 | Kolom | `tgl_approval`/`user_approval` Pengajuan diisi admin lalu ditimpa karyawan | Jejak penyetuju hilang | Kolom terpisah |
| INC-11 | Backend | Validasi request dijalankan sebelum cek admin (store, tolak, alokasi) | 422 vs 403 | Pindahkan cek akses |
| INC-12 | DB | Kolom `id_asset_kategori_m` dan `urgensi` pada header kini legacy (nullable, tidak diisi Pengajuan baru) | Baris lama masih berisi kategori/urgensi yang tidak ditampilkan lagi; skema fisik bergantung pada SQL manual v1.4 | Hapus kolom setelah data lama tidak diperlukan; pastikan SQL v1.4 dijalankan |

---

## 40. Edge Case

| Edge Case | Current Behavior | Impact | Recommendation |
|---|---|---|---|
| Karyawan melaporkan rusak | Status `proses` permanen, asset `proses` permanen | Transaksi & asset macet | Endpoint ganti/tarik/batal |
| Asset retur lalu ingin ditambahkan lagi | Diterima (v1.1): hanya header `proses` yang mengunci asset | Asset bisa dipakai ulang lewat penambahan/pengajuan | – |
| Draft penambahan / pengajuan | Dapat diedit dan dikirim (PUT); belum dapat dihapus | Draft yang tidak terpakai menumpuk di tab Pending/Pengajuan | Endpoint hapus draft |
| Dua admin mengalokasikan asset sama | Lock company; yang kedua 422 | Aman | – |
| Admin A menolak saat Admin B mengalokasikan | Lock + cek `tgl_approval`; yang kedua 422 | Aman | – |
| Karyawan nonaktif sebelum konfirmasi | 422 "Karyawan tidak ditemukan atau tidak aktif." | Asset tertahan `proses` | Aksi batal admin |
| Draft alokasi, asset diambil transaksi lain | Submit alokasi gagal "tidak tersedia" | Admin memilih ulang | – |
| Tanggal serah terima di masa lalu via API | Diterima | Data tidak valid | Validasi backend |
| Timeout tapi sudah commit | Submit ulang penambahan ditolak (asset `proses`); pengajuan karyawan bisa **dobel** | Pengajuan ganda | Idempotency / refresh |
| Banyak penambahan pending untuk satu karyawan | Banner hanya satu | Terlewat | Lihat INC-06 |
| Aplikasi lama (sebelum v1.4) mengirim pengajuan | Tanpa `jenis_asset` → 422 "Jenis asset wajib diisi."; `id_asset_kategori_m`/`urgensi` diabaikan | Pengajuan dari klien lama gagal sampai aplikasi diperbarui | Informasikan/paksa update aplikasi |
| Draft lama (dibuat sebelum v1.4) | `nama_kategori` berisi nama kategori lama dan dipertahankan saat PUT draft tanpa key `jenis_asset`; dapat dikirim lewat klien baru | Aman | – |
| SQL v1.4 belum dijalankan saat API baru aktif | INSERT pengajuan dengan `id_asset_kategori_m` NULL gagal pada kolom NOT NULL → 500 | Karyawan tidak bisa mengajukan | Jalankan `sql-pengajuan-karyawan-jenis-asset-bebas.sql` sebelum deploy |
| Jenis asset diketik dengan ejaan beragam | Tidak ada normalisasi | Sulit mengelompokkan atau menghitung per jenis | Pertimbangkan autocomplete/master jenis |

---

## 41. Potential Bug / Technical Risk

### RISK-001 — Laporan rusak membuat transaksi & asset macet (TINGGI)
**Lokasi:** `AssetRequestKaryawan_m::konfirmasiKaryawan`; tidak ada route tindak lanjut.
**Masalah:** Jika satu item rusak, status tetap `proses`, tidak ada item pending, semua asset tetap `status_asset = proses` (termasuk yang normal), dan tidak ada endpoint ganti/tarik/batal. UI admin read-only dan tidak menampilkan item rusak.
**Impact:** Transaksi tidak pernah selesai; asset tidak bisa dipakai transaksi apa pun (hanya status `tersedia` yang bisa dipilih).
**Current Handling:** Notifikasi "Laporan Asset Rusak" ke admin saja.
**Recommendation:** Implementasi alur seperti Onboarding (ganti asset rusak) atau aksi admin untuk melepas asset rusak ke gudang dan menyerahkan yang normal.

### RISK-002 — Asset bekas penambahan terkunci selamanya (TINGGI) — ✅ DIPERBAIKI v1.1
**Status:** Diperbaiki 2026-09-29. `hasActiveRequestAsset` kini hanya memfilter `status_approval = proses`. Regression test: `tests/Asset/AssetModelTest.php::test_returned_asset_from_confirmed_addition_can_be_added_again`.
**Lokasi (sebelum):** `hasActiveRequestAsset` (`whereIn status_approval [proses, dikonfirmasi]`, tanpa filter status item).
**Masalah:** Transaksi yang sudah **selesai** tetap dihitung aktif. Setelah asset dikembalikan ke gudang (Retur/Offboarding menyetel `tersedia` + `gudang`), asset muncul di pilihan tetapi `store`/`alokasi` menolak "…sedang diproses pada penambahan lain."
**Impact:** Asset tidak bisa ditambahkan ulang via Penambahan/Pengajuan.
**Recommendation:** Batasi ke `STATUS_PROSES` (dan item belum diserahkan).

### RISK-003 — Draft tidak dapat dihapus (RENDAH)
**Lokasi:** Route (ada `PUT` untuk edit/kirim draft, tetapi tidak ada `DELETE` untuk penambahan & pengajuan karyawan).
**Masalah:** Draft Penambahan (admin) dan draft Pengajuan (karyawan) kini dapat diedit dan dikirim, tetapi tidak dapat dihapus. Draft penambahan juga tidak mengunci asset.
**Recommendation:** Tambah endpoint hapus draft (pola onboarding).

### RISK-004 — `no_trx` UNIQUE global tanpa kode company
**Lokasi:** DBML `no_trx [unique]`, format `KRY-03-1-YYYYMMNNN`.
**Masalah:** Dua company pada bulan & urutan sama menghasilkan `no_trx` identik.
**Impact:** Jika unique index fisik ada, insert company kedua gagal (500). ⚠️ Asumsi: belum diverifikasi di DB.
**Recommendation:** Unique `(id_company, no_trx)`.

### RISK-005 — Tanggal nol `0000-00-00`
**Lokasi:** `store`, `storePengajuanKaryawan`, `alokasikanPengajuanAdmin`.
**Masalah:** MySQL strict mode (`NO_ZERO_DATE`) menolak nilai ini. ⚠️ Asumsi: tergantung konfigurasi SQL mode.
**Recommendation:** Kolom nullable.

### RISK-006 — Jejak approval tertimpa
`user_approval`/`tgl_approval` dari admin ditimpa karyawan saat konfirmasi (INC-10).

### RISK-007 — Jadwal lampau diterima backend
`parseJadwalValue` (validasi tanggal ≥ pengajuan) tersedia di `Asset_m` tetapi tidak dipakai fitur ini.

### RISK-008 — Tidak ada pembatasan pengajuan ganda
Karyawan dapat membuat pengajuan jenis sama berulang kali.

### RISK-009 — Kapasitas penomoran
3 digit per bulan per company, dipakai bersama Penambahan & Pengajuan (header) dan semua item.

### RISK-010 — Notifikasi di luar transaksi
Kegagalan notif hanya di-log.

### RISK-011 — Tolak menghapus draft alokasi (hard delete)
Tidak ada jejak asset yang sempat dialokasikan.

### RISK-012 — SQL v1.4 belum dijalankan sebelum deploy API (TINGGI)
Kolom `id_asset_kategori_m` masih NOT NULL di DB sampai `sql-pengajuan-karyawan-jenis-asset-bebas.sql` dijalankan, sehingga INSERT Pengajuan baru (NULL) gagal (500). Status saat ini: SQL **belum dijalankan** (langkah user, sebelum API dipakai). Cek charset/collation `nama_kategori` (`SHOW CREATE TABLE`) lebih dulu.

### RISK-013 — Jenis asset teks bebas tidak ternormalisasi (RENDAH)
Tidak ada master atau validasi isi; ejaan beragam menyulitkan laporan per jenis. Bila `nama_kategori` memakai latin1, karakter non-latin/emoji dapat gagal atau rusak (pertimbangkan utf8mb4).

---

## 42. Dependency

```text
Master Company / Data Karyawan / Asset Admin
      ↓
Master Kategori ──► Penambahan (snapshot kategori)   [Pengajuan: jenis asset teks bebas, tanpa master]
      ↓
Asset Data (tersedia @gudang)
      ↓
Request / Penambahan (header + item) ──► Notifikasi
      ↓ konfirmasi semua normal
Asset Data (terpakai, karyawan) ──► Asset Log (request_karyawan)

Bergantung pada / dipakai oleh:
  - Onboarding: hasActiveOnboardingAsset & AssetAdmin_m::getOnboardingOptions (opsi karyawan/asset)
  - Summary Admin: countPendingForAdmin / getPendingForAdmin
  - Asset Saya: getPendingForEmployee (banner penambahan)
  - Offboarding: menelusuri sumber asset dari request `dikonfirmasi` dengan item normal (jenis "onboarding")
  - Retur/Offboarding mengembalikan asset ke gudang → asset bisa ditambahkan lagi (RISK-002 diperbaiki v1.1)
  - SQL manual sql-pengajuan-karyawan-jenis-asset-bebas.sql (kolom header nullable) harus dijalankan sebelum API v1.4 dipakai
```

---

## 43. Source Code Reference

| File | Layer | Function |
|---|---|---|
| `API/routes/modules/asset/RouteAsset.php` | Route | 17 endpoint request/penambahan |
| `API/app/Http/Controllers/Asset/Admin/AssetAdminPenambahan_c.php` | Controller | Penambahan & pengajuan (admin) |
| `API/app/Http/Controllers/Asset/Karyawan/AssetKaryawanPengajuan_c.php` | Controller | Pengajuan & konfirmasi (karyawan) |
| `API/app/Models/Asset/AssetRequestKaryawan_m.php` | Model | Seluruh logika |
| `API/app/Models/Asset/Asset_m.php` | Model | Helper bersama |
| `API/app/Models/Asset/AssetAdmin_m.php` | Model | `getOnboardingOptions`, summary |
| `API/app/Models/Asset/AssetKaryawan_m.php` | Model | Asset Saya (`penambahan[]`) |
| `API/app/Models/Asset/RequestKaryawanStatus.php` | Enum | Status |
| `API/app/Support/Asset/AssetNotification.php` | Support | Event `pengajuan.*`, `penambahan.*` |
| `API/diagram-asset.dbml`; migrations `2026_09_16_000003`, `2026_09_17_000004`, `2026_09_17_000008`; SQL manual `Claude outputs/sql-pengajuan-karyawan-jenis-asset-bebas.sql` (v1.4) | DB | Struktur, index, enum, kolom nullable (v1.4) |
| `MOBILE/lib/Repository/Service/assetRequestKaryawanApi.dart`, `assetPengajuanAdminApi.dart`, `assetPengajuanKaryawanApi.dart`, `assetSayaApi.dart` | Api | Endpoint |
| `MOBILE/lib/Repository/Model/Asset/mAssetRequestKaryawan.dart`, `mAssetPengajuanKaryawan.dart`, `mAssetPengajuanAdmin.dart` | Model | – |
| `MOBILE/lib/Bloc/Asset/AssetRequestKaryawanBloc.dart`, `AssetPengajuanAdminBloc.dart`, `AssetPengajuanKaryawanBloc.dart`, `AssetKonfirmasiPenambahanBloc.dart` | Bloc | – |
| `MOBILE/lib/UI/Modul/Asset/AssetAdmin/AssetAdminKaryawan/Penambahan/*`, `Pengajuan/*` | UI admin | – |
| `MOBILE/lib/UI/Modul/Asset/AssetSaya/Pengajuan/*`, `Onboarding/HalamanKonfirmasiPenambahanAsset.dart`, `HalamanAssetSaya.dart` | UI karyawan | – |
| `MOBILE/lib/Setting/router.dart`, `Bloc/NavigatorBloc.dart` | Navigasi | – |
| `MOBILE/assets/id/asset.json`, `assets/en/asset.json` | i18n | Label |

---

## 44. Developer Notes

- **Satu tabel dua alur**: selalu filter `sumber_pengajuan`. `getDetail(..., sumber = null)` sengaja dipakai konfirmasi karyawan agar kedua sumber bisa dibuka.
- **Tahap pengajuan tanpa kolom tahap**: dibedakan oleh `status_approval`, `tgl_approval` (NULL = belum disetujui), dan jumlah item (ada item + belum disetujui = Draft Alokasi).
- **Label dihitung di tiga fungsi berbeda** (`requestStatusMetadata`, `pengajuanAdminStatusMetadata`, `pengajuanStatusMetadata`) dan dipetakan ulang di mobile — ubah bersamaan.
- **Karyawan tidak melihat alokasi sebelum disetujui**: `mapPengajuanKaryawanRecords` & `getDetailPengajuanKaryawan` mengosongkan jadwal/item.
- **Opsi asset memakai fungsi onboarding** (`AssetAdmin_m::getOnboardingOptions` dengan `excludeEmployeesWithOnboarding: false`).
- **Jenis asset Pengajuan = teks bebas (v1.4)**: `nama_kategori` menyimpan teks; `id_asset_kategori_m` dan `urgensi` NULL (baris lama tetap berisi kategori/urgensi). Jangan memfilter alokasi dengan `id_asset_kategori_m` header.
- **Sisa `urgensi` di kode**: `store()` Penambahan admin masih menulis `urgensi = ''` (kolom nullable) dan mobile `AssetRequestKaryawanRequest` masih mengirim `''`; dibiarkan agar kontrak Penambahan tidak berubah.
- **Reservasi hanya saat kirim**; draft tidak mengunci asset (beda dengan onboarding).
- **Jam** dikirim `H:i`, disimpan `H:i:s`; tanggal kosong disimpan `0000-00-00` dan diabaikan mobile lewat `hasSchedule`.
- **`asset_trx_log.jenis_log = request_karyawan`** — dipakai Offboarding untuk menelusuri asal asset.
- **Technical debt**: RISK-001/003 (RISK-002 sudah diperbaiki), enum ganda, `parent_id`/`diganti` tidak terpakai.

---

## 45. End-to-End Traceability

### 45.1 Pengajuan → jenis asset
```text
Karyawan mengetik "Laptop" di field Jenis Asset
  → _categoryController.text.trim()
  → AssetPengajuanKaryawanRequest.jenisAsset
  → JSON "jenis_asset"
  → AssetKaryawanPengajuan_c::storePengajuanSaya (trim, required saat kirim, max:150)
  → AssetRequestKaryawan_m::storePengajuanKaryawan
  → asset_trx_request_karyawan.nama_kategori = "Laptop" (id_asset_kategori_m = NULL, urgensi = NULL)
  → (admin) getPengajuanAdminOptions: seluruh stok tersedia, tanpa filter kategori
```

### 45.2 Alokasi → asset diterima karyawan
```text
Admin memilih asset di Setujui Pengajuan Asset
  → _selectedAssets[].idAssetData → AssetPengajuanAdminAlokasiRequest.assetIds
  → JSON "asset_ids"
  → alokasikanPengajuanAdmin: tersedia @gudang (tanpa filter kategori)
  → INSERT asset_trx_request_karyawan_item (belum_dicek)
  → UPDATE asset_data.status_asset = proses
Karyawan memilih Normal → statusByItem → JSON items[].status_item
  → konfirmasiKaryawan: semua normal
  → UPDATE asset_data: terpakai, pemegang_asset = karyawan, id_pemegang_asset = id_karyawan
  → INSERT asset_trx_log (request_karyawan)
  → asset tampil di Asset Saya
```

### 45.3 DB → UI
```text
asset_trx_request_karyawan.ket_tolak
  → getDetail → API "ket_tolak"
  → AssetRequestKaryawanRecord.ketTolak
  → UI "Alasan Penolakan" (detail admin & karyawan)
```

### 45.4 Walkthrough Lengkap dengan Contoh Data

Bagian ini mengikuti **dua transaksi dari awal sampai selesai**, langkah demi langkah: apa yang dilakukan pengguna di mobile, apa yang dikirim, apa yang diproses API, dan apa yang tersimpan di database. Data bersifat **sintetis**, tetapi format ID, nama kolom, urutan proses, label status, dan pesan diambil dari kode (`AssetRequestKaryawan_m`, `AssetAdminPenambahan_c`, `AssetAdminPengajuan_c`, `AssetKaryawanPengajuan_c`, `AssetKaryawanPenambahan_c`, `AssetNotification`).

Dua jalur memakai **tabel yang sama** (`asset_trx_request_karyawan` + `_item`); pembedanya `sumber_pengajuan`.

| Skenario | Jalur | Ringkasan |
|---|---|---|
| **A. Penambahan** | Admin → Karyawan (`sumber_pengajuan = admin`) | Admin memberi laptop dan mouse kepada Sari: draft, publish, Sari mengonfirmasi semua Normal |
| **B. Pengajuan** | Karyawan → Admin → Karyawan (`sumber_pengajuan = karyawan`) | Dewi mengajukan monitor: draft, kirim, admin mengalokasikan (draft lalu kirim), Dewi mengonfirmasi |

#### Data awal

Company `CMP001`. Tanggal kejadian 22 Oktober 2026, periode ID `202610`. Belum ada transaksi request, log, atau notifikasi pada periode ini.

**`data_karyawan`**

| id_karyawan | nik | nama_lengkap | Peran |
|---|---|---|---|
| KRY0099 | ADM-001 | Siti Admin | Admin Asset (aktor) |
| KRY0100 | ADM-002 | Rudi Admin | Admin Asset lain (ikut menerima notifikasi) |
| KRY0002 | EMP-002 | Sari Contoh | Penerima Penambahan (skenario A) |
| KRY0003 | EMP-003 | Dewi Contoh | Pemohon Pengajuan (skenario B) |

KRY0099 dan KRY0100 terdaftar di `asset_admin`.

**`asset_data`** (semua `status_asset = tersedia`, `pemegang_asset = gudang`, `id_pemegang_asset = ''`, `kondisi_awal = normal`)

| Kode | id_asset_data | nama_asset | kategori | deskripsi |
|---|---|---|---|---|
| A5 | CMP00120260100005 (LPT-0005) | Laptop 14 inch | Laptop (`CMP001202601002`) | Charger, tas |
| A6 | CMP00120260100006 (MSE-0002) | Mouse Wireless | Aksesoris (`CMP001202601003`) | `''` |
| A7 | CMP00120260100007 (MON-0001) | Monitor 24 inch | Monitor (`CMP001202601004`) | Kabel HDMI |

Penomoran ID bersama untuk kedua jalur: header `id_company + YYYYMM + 3 digit`, item `id_company + YYYYMM + 3 digit`, `no_trx = KRY-03-1-YYYYMMNNN`, log dan notifikasi 5 digit.

---

### Skenario A — Penambahan oleh Admin

#### Langkah A0 — Admin membuka form dan memuat opsi

**Mobile:** route `/asset_admin_add_penambahan_karyawan` → `LoadAssetRequestKaryawanOptions` → `AssetRequestKaryawanApi.getOptions()`.

**Dikirim**
```http
GET /asset/penambahan-karyawan/options
Authorization: Bearer <access_token_admin>
```

**Diproses API:** `jwt.api` + `jwt.ctx` → `accessError()` (harus admin asset) → `AssetRequestKaryawan_m::getOptions` → `AssetAdmin_m::getOnboardingOptions(excludeEmployeesWithOnboarding: false)`. Berbeda dari Onboarding, **semua karyawan aktif** muncul (termasuk yang sudah pernah di-onboarding). Asset yang tampil: `tersedia` + `gudang`.

**Database:** hanya `SELECT`.

**Diterima mobile:** `karyawan[]`, `kategori_asset[]` (pohon), `assets[]` (A5, A6, A7) seperti API-001. Dipakai untuk bottom sheet **Pilih Karyawan** dan pohon **Tambah Asset**.

#### Langkah A1 — Admin menyimpan draft (1 asset)

**Mobile:** admin memilih Sari Contoh, menambah **Laptop 14 inch**, mengisi jadwal 29 Okt 2026 pukul 09:00 dan catatan, lalu **Simpan Draft** → `AssetRequestKaryawanApi.store(AssetRequestKaryawanRequest)` dengan `isDraft: true`.

**Dikirim**
```http
POST /asset/penambahan-karyawan
```
```json
{
  "id_karyawan": "KRY0002",
  "asset_ids": ["CMP00120260100005"],
  "tgl_serah_terima": "2026-10-29",
  "jam_serah_terima": "09:00",
  "keterangan_serah_terima": "Ambil di ruang IT",
  "keterangan": "",
  "urgensi": "",
  "is_draft": true
}
```
(`urgensi` legacy, mobile masih mengirim `''`.)

**Diproses API** (`AssetAdminPenambahan_c::savePenambahanKaryawan` → `AssetRequestKaryawan_m::store`, satu `DB::transaction`)
1. Validasi request, lalu `accessError()`.
2. `lockCompany("CMP001")`.
3. `requestLockedAssets`: A5 harus `tersedia` + `gudang` (`FOR UPDATE`); jumlah harus sama.
4. `hasActiveOnboardingAsset` dan `hasActiveRequestAsset` (hanya header `proses` yang mengunci): lolos.
5. `findActiveKaryawan("KRY0002")` dan `findAssetAdmin("KRY0099")` → `user_input = "ADM-001 - Siti Admin"`.
6. Karena `is_draft = true`, jadwal boleh kosong (di sini terisi).
7. Generate ID header: urutan 1 → `CMP001202610001`, `no_trx = KRY-03-1-202610001`.
8. Kategori: semua asset satu kategori → `id_asset_kategori_m = CMP001202601002`, `nama_kategori = Laptop`.
9. INSERT header (`draft`), INSERT item.
10. **Draft tidak mengunci asset** dan tidak mengirim notifikasi.

**Hasil database**

`asset_trx_request_karyawan`

| Kolom | Nilai |
|---|---|
| id_asset_trx_request_karyawan | CMP001202610001 |
| no_trx | KRY-03-1-202610001 |
| sumber_pengajuan | **admin** |
| id_asset_kategori_m / nama_kategori | CMP001202601002 / Laptop |
| keterangan / urgensi / ket_tolak | `''` / `''` / `''` |
| id_karyawan / nik_karyawan / nama_karyawan | KRY0002 / EMP-002 / Sari Contoh |
| tgl_pengajuan | 2026-10-22 |
| tgl_serah_terima / jam_serah_terima | 2026-10-29 / 09:00:00 |
| keterangan_serah_terima | Ambil di ruang IT |
| status_approval | **draft** |
| user_approval / tgl_approval | `''` / NULL |
| tgl_input / user_input | 2026-10-22 09:00:00 / ADM-001 - Siti Admin |
| tgl_update / user_update | NULL / `''` |

`asset_trx_request_karyawan_item`

| id_asset_trx_request_item | parent_id | id_asset_trx_request | id_asset_data | nama_asset | deskripsi_asset | kategori_asset | status_item |
|---|---|---|---|---|---|---|---|
| CMP001202610001 | `''` | CMP001202610001 | A5 | Laptop 14 inch | Charger, tas | Laptop | belum_dicek |

`asset_data`: **tidak berubah** (A5 tetap `tersedia`).

**Diterima mobile** (HTTP 201)
```json
{
  "code": "CREATED",
  "message": "Draft penambahan asset berhasil disimpan.",
  "data": { "id": "CMP001202610001", "no_trx": "KRY-03-1-202610001",
            "status_approval": "draft", "status_label": "Draft", "tab": "pending" }
}
```

#### Langkah A2 — Admin menambah mouse lalu publish

**Mobile:** admin membuka draft (kartu tab **Pending**), menambah **Mouse Wireless**, menekan **Submit ke Karyawan** → `AssetRequestKaryawanApi.update(id, request)`.

**Dikirim**
```http
PUT /asset/penambahan-karyawan/CMP001202610001
```
```json
{
  "id_karyawan": "KRY0002",
  "asset_ids": ["CMP00120260100005", "CMP00120260100006"],
  "tgl_serah_terima": "2026-10-29",
  "jam_serah_terima": "09:00",
  "keterangan_serah_terima": "Ambil di ruang IT",
  "keterangan": "",
  "urgensi": "",
  "is_draft": false
}
```

**Diproses API** (`store(data, id)` dengan `$idRequest` terisi)
1. Header dicari dengan `sumber_pengajuan = admin`, `FOR UPDATE`; harus `draft`, selain itu 422 "Hanya draft penambahan langsung yang dapat diperbarui."
2. Validasi asset (A5 dan A6 `tersedia` + `gudang`, tidak aktif di onboarding/penambahan lain), karyawan, admin. Jadwal **wajib** karena `is_draft = false`.
3. ID header dan `no_trx` **dipertahankan**; `tgl_pengajuan` dan `tgl_input` juga dipertahankan.
4. Kategori: dua kategori berbeda (Laptop dan Aksesoris) → `id_asset_kategori_m = ''`, `nama_kategori = 'Multi Kategori'`.
5. UPDATE header → `proses`; `nextSequence` item dihitung **sebelum** item lama dihapus (terakhir `…001` → mulai `2`), lalu DELETE item lama dan INSERT item baru.
6. Karena bukan draft: UPDATE `asset_data` A5 dan A6 → `proses`.
7. Controller memanggil `AssetNotification::send('penambahan.dikirim', …)` setelah commit.

**Hasil database**

Header (kolom yang berubah)

| Kolom | Sebelum | Sesudah |
|---|---|---|
| id_asset_kategori_m / nama_kategori | CMP001202601002 / Laptop | `''` / Multi Kategori |
| status_approval | draft | **proses** |
| tgl_approval | NULL | NULL |
| tgl_update / user_update | NULL / `''` | 2026-10-22 09:10:00 / ADM-001 - Siti Admin |

Item: baris `…001` **dihapus** dan diganti (ID lama tidak dipakai ulang):

| id_asset_trx_request_item | id_asset_data | nama_asset | deskripsi_asset | kategori_asset | status_item |
|---|---|---|---|---|---|
| CMP001202610002 | A5 | Laptop 14 inch | Charger, tas | Laptop | belum_dicek |
| CMP001202610003 | A6 | Mouse Wireless | `''` | Aksesoris | belum_dicek |

`asset_data`: A5 dan A6 `status_asset` **tersedia → proses** (`tgl_update` 09:10:00, `user_update` ADM-001 - Siti Admin).

`notif_data` (1 baris)

| Kolom | Nilai |
|---|---|
| id_notif | CMP00120261000001 |
| id_karyawan | KRY0002 |
| jenis / flag_baca | asset / 0 |
| id_ref | CMP001202610001 |
| rute | asset_konfirmasi_penambahan |
| ket_notif_id | Anda menerima penambahan asset KRY-03-1-202610001. |
| ket_notif_en | You received asset addition KRY-03-1-202610001. |

**Diterima mobile** (HTTP 200)
```json
{
  "code": "UPDATED",
  "message": "Penambahan asset berhasil dikirim ke karyawan.",
  "data": { "id": "CMP001202610001", "no_trx": "KRY-03-1-202610001",
            "status_approval": "proses", "status_label": "Menunggu Diterima", "tab": "pending" }
}
```

#### Langkah A3 — Sari mengonfirmasi: semua Normal

**Mobile**
- Push membuka `/asset_konfirmasi_penambahan` → `LoadAssetKonfirmasiPenambahan(id)` → `GET /asset/karyawan/penambahan/CMP001202610001` (API-014). Detail berisi 2 item `belum_dicek`, `perlu_dicek: true`, jadwal serah terima, dan `karyawan` (nik, nama, jabatan, cabang, divisi).
- Sari memilih **Normal** pada keduanya, **Konfirmasi Asset** → `SubmitAssetKonfirmasiPenambahan` → `AssetSayaApi.konfirmasiPenambahan`.

**Dikirim**
```http
POST /asset/karyawan/penambahan/CMP001202610001/konfirmasi
```
```json
{
  "items": [
    { "id_item": "CMP001202610002", "status_item": "normal" },
    { "id_item": "CMP001202610003", "status_item": "normal" }
  ]
}
```
`ket_rusak` hanya ikut dikirim untuk item rusak.

**Diproses API** (`AssetKaryawanPenambahan_c::konfirmasiPenambahanSaya` → `AssetRequestKaryawan_m::konfirmasiKaryawan`)
1. Validasi `items` (id unik, `status_item` ∈ normal/rusak, `ket_rusak` wajib bila rusak).
2. `lockCompany`; header diambil dengan `id_karyawan = KRY0002` + `FOR UPDATE`; harus `proses`. Untuk Pengajuan, `tgl_approval` wajib terisi (jalur ini `admin`, jadi tidak diperiksa).
3. Karyawan harus aktif. Item dikunci; item pending = `belum_dicek` = `…002` dan `…003`; jumlah dan ID yang dikirim harus sama persis.
4. UPDATE item: `status_item = normal`, `ket_rusak = ''`.
5. `semuaNormal` dihitung atas **seluruh item** → true → status `dikonfirmasi`.
6. UPDATE header: `status_approval`, `user_approval`, `tgl_approval = now`, `tgl_update`, `user_update`.
7. **Karena semua normal**: asset dikunci, UPDATE `asset_data` → `terpakai`, `karyawan`, `id_pemegang_asset = KRY0002`, `id_asset_lokasi_m = NULL`; INSERT `asset_trx_log` satu baris per asset.
8. Controller memanggil `AssetNotification::send('penambahan.dikonfirmasi', …)` ke seluruh admin.

**Hasil database**

| Tabel | Perubahan |
|---|---|
| item `…002`, `…003` | status_item `belum_dicek → normal`, ket_rusak `''` |
| header | status_approval `proses → dikonfirmasi`; user_approval = EMP-002 - Sari Contoh; tgl_approval = 2026-10-22 10:00:00; tgl_update/user_update diisi yang sama |
| asset_data A5, A6 | status_asset `proses → terpakai`; pemegang_asset `gudang → karyawan`; id_pemegang_asset `'' → KRY0002`; id_asset_lokasi_m → NULL |

`asset_trx_log` (2 baris, urutan sesuai `id_asset_data` pada transaksi)

| id_asset_trx_log | id_asset_data | jenis_log | id_transaksi | nama_tabel_transaksi | kondisi awal → akhir | posisi awal → akhir | id_pemegang / nama_pemegang | user_input |
|---|---|---|---|---|---|---|---|---|
| CMP00120261000001 | A5 | request_karyawan | CMP001202610001 | asset_trx_request_karyawan | normal → normal | gudang → karyawan | KRY0002 / Sari Contoh | EMP-002 - Sari Contoh |
| CMP00120261000002 | A6 | request_karyawan | CMP001202610001 | asset_trx_request_karyawan | normal → normal | gudang → karyawan | KRY0002 / Sari Contoh | EMP-002 - Sari Contoh |

`notif_data` (2 baris, satu per admin)

| id_notif | id_karyawan | id_ref | rute | ket_notif_id |
|---|---|---|---|---|
| CMP00120261000002 | KRY0099 | CMP001202610001 | asset_admin_detail_penambahan_karyawan | Asset KRY-03-1-202610001 telah dikonfirmasi oleh Sari Contoh. |
| CMP00120261000003 | KRY0100 | CMP001202610001 | asset_admin_detail_penambahan_karyawan | Asset KRY-03-1-202610001 telah dikonfirmasi oleh Sari Contoh. |

Rute admin dipilih dari `sumber_pengajuan` header (`admin` → detail penambahan, `karyawan` → detail pengajuan).

**Diterima mobile** (HTTP 200)
```json
{
  "code": "UPDATED",
  "message": "Penambahan asset berhasil dikonfirmasi.",
  "data": { "id": "CMP001202610001", "no_trx": "KRY-03-1-202610001",
            "status_approval": "dikonfirmasi", "status_label": "Selesai", "tab": "selesai" }
}
```
Laptop dan mouse kini muncul di Asset Saya Sari.

#### Variasi A — ada asset rusak (all-or-nothing)

Bila pada langkah A3 laptop dinilai **Rusak** ("Ada garis pada layar") dan mouse **Normal**:

```json
{ "items": [
  { "id_item": "CMP001202610002", "status_item": "rusak", "ket_rusak": "Ada garis pada layar" },
  { "id_item": "CMP001202610003", "status_item": "normal" } ] }
```

| Tabel | Hasil |
|---|---|
| item `…002` / `…003` | `rusak` + ket_rusak / `normal` |
| header | status_approval **tetap proses**; `tgl_approval` = waktu konfirmasi; user_approval = EMP-002 - Sari Contoh |
| asset_data A5 **dan** A6 | **Tidak berubah**: keduanya tetap `proses` di gudang (mouse yang Normal pun belum diserahkan) |
| asset_trx_log | **Tidak ada baris** |
| notif_data | `penambahan.rusak` ke setiap admin: "Sari Contoh melaporkan asset rusak pada KRY-03-1-202610001." |

Respons: message "Laporan kerusakan penambahan asset berhasil dikirim.", `status_label = "Isu Rusak"`, `tab = "dikonfirmasi"`. Tidak ada endpoint lanjutan, sehingga kedua asset tertahan di `proses` (RISK-001). Ini berbeda dari Onboarding yang menyerahkan item Normal per item dan menyediakan penggantian.

---

### Skenario B — Pengajuan oleh Karyawan

#### Langkah B1 — Dewi menyimpan draft pengajuan

**Mobile:** Asset Saya → **Pengajuan** → (+) → route `/asset_tambah_pengajuan`. Dewi mengetik **Jenis Asset** (teks bebas, maks 150) dan **Alasan**, lalu **Simpan Draft** → `AssetPengajuanKaryawanApi.store(AssetPengajuanKaryawanRequest)`.

**Dikirim**
```http
POST /asset/karyawan/pengajuan
Authorization: Bearer <access_token_dewi>
```
```json
{ "jenis_asset": "Monitor 24 inch", "keterangan": "Monitor lama bergaris dan sering mati", "is_draft": true }
```

**Diproses API** (`AssetKaryawanPengajuan_c::storePengajuanSaya` → `AssetRequestKaryawan_m::storePengajuanKaryawan`)
1. Validasi: `jenis_asset` (string, maks 150) dan `keterangan` wajib hanya bila bukan draft; `is_draft` boolean. Field `id_asset_kategori_m` dan `urgensi` dari klien lama diabaikan.
2. `lockCompany`; `findActiveKaryawan("KRY0003")` (tanpa cek admin: pengguna biasa).
3. Generate ID: `nextSequence` header → urutan 2 (urutan 1 dipakai skenario A) → `CMP001202610002`, `no_trx = KRY-03-1-202610002`.
4. INSERT header saja (**tanpa item**); `nama_kategori` = teks jenis asset; `id_asset_kategori_m` dan `urgensi` = NULL (butuh SQL v1.4 agar kolom nullable).
5. Draft: tidak ada notifikasi.

**Hasil database** — `asset_trx_request_karyawan` (satu baris baru)

| Kolom | Nilai |
|---|---|
| id_asset_trx_request_karyawan | CMP001202610002 |
| no_trx | KRY-03-1-202610002 |
| sumber_pengajuan | **karyawan** |
| id_asset_kategori_m / urgensi | **NULL** / NULL |
| nama_kategori | Monitor 24 inch (teks bebas dari karyawan) |
| keterangan | Monitor lama bergaris dan sering mati |
| id_karyawan / nik_karyawan / nama_karyawan | KRY0003 / EMP-003 / Dewi Contoh |
| tgl_pengajuan | 2026-10-22 |
| tgl_serah_terima / jam_serah_terima / keterangan_serah_terima | 0000-00-00 / 00:00:00 / `''` |
| status_approval | **draft** |
| user_approval / tgl_approval | `''` / NULL |
| tgl_input / user_input | 2026-10-22 11:00:00 / EMP-003 - Dewi Contoh |

`asset_trx_request_karyawan_item` dan `asset_data`: tidak berubah.

**Diterima mobile** (HTTP 201)
```json
{ "code": "CREATED", "message": "Draft pengajuan asset berhasil disimpan.",
  "data": { "id": "CMP001202610002", "no_trx": "KRY-03-1-202610002",
            "status_approval": "draft", "status_label": "Draft", "tab": "pengajuan" } }
```

#### Langkah B2 — Dewi mengirim pengajuan ke admin

**Mobile:** kartu draft di tab **Pengajuan** → `LoadAssetPengajuanKaryawanDraft(id)` mengisi form → **Submit ke Admin** → `AssetPengajuanKaryawanApi.update(id, request)`.

**Dikirim**
```http
PUT /asset/karyawan/pengajuan/CMP001202610002
```
```json
{ "jenis_asset": "Monitor 24 inch", "keterangan": "Monitor lama bergaris dan sering mati", "is_draft": false }
```

**Diproses API** (`updatePengajuanKaryawan`)
1. Header dicari dengan `id_karyawan = KRY0003` dan `sumber_pengajuan = karyawan` (bukan milik pemanggil → 404 "Pengajuan asset tidak ditemukan.").
2. Syarat draft: `status_approval = draft`, `tgl_approval` NULL, `user_approval` kosong, belum punya item; selain itu 422 "Pengajuan asset sudah tidak dapat diubah karena bukan draft."
3. Saat kirim, `jenis_asset` dan `keterangan` wajib.
4. UPDATE header: `keterangan`, `urgensi = NULL`, `status_approval = proses`, `tgl_update`, `user_update`, `tgl_pengajuan = hari ini`; `nama_kategori` ditimpa bila key `jenis_asset` dikirim.
5. Controller memanggil `AssetNotification::send('pengajuan.diajukan', …)` ke seluruh admin.

**Hasil database**

| Tabel | Perubahan |
|---|---|
| header | status_approval `draft → proses`; tgl_pengajuan 2026-10-22; tgl_update 11:05:00; user_update = EMP-003 - Dewi Contoh; tgl_approval tetap NULL |
| item, asset_data | Tidak berubah |
| notif_data | + `CMP00120261000004` (KRY0099) dan `CMP00120261000005` (KRY0100), rute `asset_admin_detail_pengajuan_karyawan`, id_ref `CMP001202610002`, ket_notif_id "Dewi Contoh mengajukan asset KRY-03-1-202610002." |

**Diterima mobile** (HTTP 200)
```json
{ "code": "UPDATED", "message": "Pengajuan asset berhasil dikirim ke admin.",
  "data": { "id": "CMP001202610002", "no_trx": "KRY-03-1-202610002",
            "status_approval": "proses", "status_label": "Menunggu Konfirmasi", "tab": "pengajuan" } }
```
Dari sudut pandang admin, label transaksi ini **"Menunggu Diterima"**, tab `pengajuan` (belum disetujui, tanpa item).

#### Langkah B3 — Admin meninjau pengajuan

**Mobile:** notifikasi membuka `/asset_admin_detail_pengajuan_karyawan` → `LoadAssetPengajuanAdminDetail(id)` → `GET /asset/pengajuan-karyawan/CMP001202610002`. Daftar (`GET /asset/pengajuan-karyawan?tab=pengajuan`) menampilkan transaksi `proses` yang belum disetujui; draft karyawan tidak pernah tampil di sisi admin.

**Diproses API:** `getDetailPengajuanAdmin`, hanya `SELECT`. Jenis asset dibaca dari `nama_kategori` dan dikirim sebagai `nama_kategori` serta `nama_asset`; `id_asset_kategori_m` = `''`.

Tombol **Tolak** dan **Setujui** tampil karena status `proses` dan tab `pengajuan`.

#### Langkah B4 — Admin menyimpan draft alokasi

**Mobile:** **Setujui** → route `/asset_admin_setujui_pengajuan_karyawan` → `LoadAssetPengajuanAdminOptions(id)` → `GET /asset/pengajuan-karyawan/CMP001202610002/options` (seluruh stok gudang, tanpa filter kategori; hanya bila pengajuan `proses` dan belum disetujui, selain itu 404). Admin memilih **Monitor 24 inch (A7)**, mengisi jadwal, lalu **Simpan Draft** → `AssetPengajuanAdminApi.allocate`.

**Dikirim**
```http
POST /asset/pengajuan-karyawan/CMP001202610002/alokasi
```
```json
{ "asset_ids": ["CMP00120260100007"], "tgl_serah_terima": "2026-10-30", "jam_serah_terima": "10:00",
  "keterangan_serah_terima": "Serah terima di ruang IT lantai 2", "is_draft": true }
```

**Diproses API** (`alokasikanPengajuanAdmin`)
1. Validasi dan cek admin; `lockCompany`.
2. Header (sumber karyawan, `FOR UPDATE`) harus `proses` **dan** `tgl_approval` NULL; selain itu 422 "Pengajuan asset sudah tidak dapat dialokasikan."
3. A7 harus `tersedia` + `gudang`; tidak aktif di onboarding atau penambahan lain (transaksi ini dikecualikan). Jenis asset yang diajukan tidak dibandingkan dengan kategori A7.
4. Jadwal boleh kosong untuk draft.
5. `nextSequence` item → 4; DELETE item lama (tidak ada), INSERT item.
6. UPDATE header: jadwal, `ket_tolak = ''`, `user_approval = ''`, `tgl_approval = NULL` (draft), `tgl_update`, `user_update`.
7. Draft: `asset_data` **tidak diubah**, tidak ada notifikasi.

**Hasil database**

| Tabel | Perubahan |
|---|---|
| item (baru) | `CMP001202610004`, id_asset_trx_request = `CMP001202610002`, id_asset_data = A7, nama_asset = Monitor 24 inch, deskripsi_asset = Kabel HDMI, kategori_asset = Monitor, status_item = belum_dicek, user_input = ADM-001 - Siti Admin |
| header | tgl_serah_terima 2026-10-30; jam_serah_terima 10:00:00; keterangan_serah_terima = Serah terima di ruang IT lantai 2; tgl_update 13:00:00; user_update = ADM-001 - Siti Admin; **status_approval tetap proses, tgl_approval tetap NULL** |
| asset_data A7 | Tidak berubah (`tersedia`) |

**Diterima mobile** (HTTP 200)
```json
{ "code": "UPDATED", "message": "Draft alokasi pengajuan asset berhasil disimpan.",
  "data": { "id": "CMP001202610002", "no_trx": "KRY-03-1-202610002", "status_approval": "proses",
            "status_label": "Draft Alokasi", "tab": "pengajuan", "jumlah_asset": 1 } }
```
Dewi **belum** melihat jadwal atau asset: untuk karyawan, sebelum disetujui `items = []`, jadwal kosong, `jumlah_asset = 0`.

#### Langkah B5 — Admin mengirim alokasi (Submit ke PIC)

**Dikirim**
```http
POST /asset/pengajuan-karyawan/CMP001202610002/alokasi
```
```json
{ "asset_ids": ["CMP00120260100007"], "tgl_serah_terima": "2026-10-30", "jam_serah_terima": "10:00",
  "keterangan_serah_terima": "Serah terima di ruang IT lantai 2", "is_draft": false }
```

**Diproses API:** sama dengan B4, dengan perbedaan:
- jadwal **wajib** (kosong → 422 "Tanggal serah terima wajib diisi." / "Jam serah terima wajib diisi.");
- `nextSequence` item → terakhir `…004` → **5**, item lama `…004` dihapus lalu diganti `…005`;
- header: `user_approval = ADM-001 - Siti Admin`, `tgl_approval = now` (pengajuan **dianggap disetujui**);
- UPDATE `asset_data` A7 → `proses`;
- `AssetNotification::send('pengajuan.disetujui', …)`.

**Hasil database**

| Tabel | Perubahan |
|---|---|
| item | `CMP001202610004` dihapus; `CMP001202610005` (A7, belum_dicek) |
| header | user_approval = ADM-001 - Siti Admin; **tgl_approval = 2026-10-22 13:30:00**; status_approval tetap `proses` |
| asset_data A7 | status_asset `tersedia → proses` |
| notif_data | + `CMP00120261000006`: id_karyawan KRY0003, id_ref `CMP001202610002`, rute `asset_konfirmasi_pengajuan`, ket_notif_id "Pengajuan asset KRY-03-1-202610002 disetujui. Silakan cek dan konfirmasi asset." |

**Diterima mobile** (HTTP 200, code `SUCCESS` karena bukan draft)
```json
{ "code": "SUCCESS", "message": "Pengajuan asset berhasil disetujui.",
  "data": { "id": "CMP001202610002", "no_trx": "KRY-03-1-202610002", "status_approval": "proses",
            "status_label": "Menunggu Konfirmasi", "tab": "proses", "jumlah_asset": 1 } }
```

#### Langkah B6 — Dewi mengonfirmasi asset

**Mobile:** push membuka `/asset_konfirmasi_pengajuan` (`isPengajuan: true`) → `LoadAssetKonfirmasiPenambahan(id)` → `GET /asset/karyawan/penambahan/CMP001202610002`. Sekarang item, jadwal, jenis asset, dan alasan tampil. Dewi memilih **Normal** → **Konfirmasi Asset**.

**Dikirim**
```http
POST /asset/karyawan/penambahan/CMP001202610002/konfirmasi
```
```json
{ "items": [ { "id_item": "CMP001202610005", "status_item": "normal" } ] }
```
Endpoint konfirmasi **sama** untuk kedua jalur.

**Diproses API:** `konfirmasiKaryawan` seperti langkah A3, ditambah syarat Pengajuan: `tgl_approval` harus terisi, jika tidak 422 "Pengajuan asset belum disubmit oleh admin."

**Hasil database**

| Tabel | Perubahan |
|---|---|
| item `…005` | status_item `belum_dicek → normal` |
| header | status_approval `proses → dikonfirmasi`; user_approval = EMP-003 - Dewi Contoh (**menimpa** nama admin); tgl_approval = 2026-10-22 15:00:00 (**menimpa** waktu persetujuan admin) |
| asset_data A7 | `proses → terpakai`; pemegang_asset `gudang → karyawan`; id_pemegang_asset = KRY0003; id_asset_lokasi_m → NULL |
| asset_trx_log | + `CMP00120261000003`: A7, jenis_log request_karyawan, id_transaksi `CMP001202610002`, posisi gudang → karyawan, id_pemegang KRY0003, nama_pemegang Dewi Contoh, user_input EMP-003 - Dewi Contoh |
| notif_data | + `CMP00120261000007` (KRY0099) dan `CMP00120261000008` (KRY0100), rute `asset_admin_detail_pengajuan_karyawan`, "Asset KRY-03-1-202610002 telah dikonfirmasi oleh Dewi Contoh." |

**Diterima mobile** (HTTP 200): `status_approval: "dikonfirmasi"`, `status_label: "Selesai"`, `tab: "selesai"`.

Karena `tgl_approval` dan `user_approval` ditimpa, pelaku dan waktu persetujuan admin **tidak tersimpan** setelah konfirmasi; hanya `tgl_update` pada langkah-langkah sebelumnya yang pernah memuat jejaknya.

#### Variasi B — admin menolak pengajuan

Pada langkah B3, admin menekan **Tolak** dan mengisi alasan.

```http
POST /asset/pengajuan-karyawan/CMP001202610002/tolak
```
```json
{ "ket_tolak": "Stok monitor belum tersedia bulan ini" }
```

**Diproses API** (`tolakPengajuanAdmin`): header harus `proses` dan belum disetujui; alasan wajib (maks 255); **DELETE** seluruh item (draft alokasi bila ada); UPDATE header.

| Tabel | Hasil |
|---|---|
| item | Dihapus semua |
| header | status_approval = **ditolak**; ket_tolak = Stok monitor belum tersedia bulan ini; user_approval = ADM-001 - Siti Admin; tgl_approval = waktu tolak; tgl_update/user_update diisi |
| asset_data | Tidak berubah (tidak pernah dikunci) |
| notif_data | + 1 baris ke KRY0003: rute `asset_detail_pengajuan`, "Pengajuan asset KRY-03-1-202610002 ditolak admin." |

Respons: code `UPDATED`, "Pengajuan asset berhasil ditolak.", `status_label = "Ditolak"`, `tab = "selesai"` (di sisi karyawan tab `ditolak`). Status `ditolak` bersifat final.

---

### Kondisi akhir data

`asset_trx_request_karyawan`

| id | no_trx | sumber | nama_kategori | status_approval | user_approval | tgl_approval |
|---|---|---|---|---|---|---|
| CMP001202610001 | KRY-03-1-202610001 | admin | Multi Kategori | dikonfirmasi | EMP-002 - Sari Contoh | 2026-10-22 10:00:00 |
| CMP001202610002 | KRY-03-1-202610002 | karyawan | Monitor 24 inch | dikonfirmasi | EMP-003 - Dewi Contoh | 2026-10-22 15:00:00 |

`asset_trx_request_karyawan_item`

| id | header | asset | status_item |
|---|---|---|---|
| CMP001202610002 | CMP001202610001 | A5 Laptop | normal |
| CMP001202610003 | CMP001202610001 | A6 Mouse | normal |
| CMP001202610005 | CMP001202610002 | A7 Monitor | normal |

ID item `…001` dan `…004` pernah ada dan sudah dihapus; nomornya **tidak dipakai ulang** (menggerus kapasitas 999/bulan, RISK-004).

`asset_data`: A5, A6 → `terpakai`/`karyawan`/`KRY0002`; A7 → `terpakai`/`karyawan`/`KRY0003`. `asset_trx_log`: 3 baris. `notif_data`: 8 baris (1 penambahan dikirim, 2 dikonfirmasi, 2 pengajuan diajukan, 1 disetujui, 2 dikonfirmasi).

---

### Kasus gagal yang umum

| Situasi | Langkah | HTTP | Pesan | Perubahan database |
|---|---|:---:|---|---|
| Asset bukan `tersedia` + `gudang` | A1 / A2 / B4 / B5 | 422 | Satu atau beberapa asset tidak tersedia untuk penambahan. / …tidak tersedia. | Tidak ada |
| Asset ada di onboarding aktif | A1 / A2 / B4 / B5 | 422 | Satu atau beberapa asset sedang diproses pada onboarding. | Tidak ada |
| Asset ada di penambahan/pengajuan `proses` lain | A1 / A2 / B4 / B5 | 422 | Satu atau beberapa asset sedang diproses pada penambahan lain. | Tidak ada |
| Kirim penambahan tanpa jadwal | A2 / B5 | 422 | Tanggal serah terima wajib diisi. / Jam serah terima wajib diisi. | Tidak ada |
| Format jam bukan `H:i` | A1 / A2 / B4 / B5 | 422 | Format jam serah terima harus HH:MM. | Tidak ada |
| Edit penambahan yang bukan draft / bukan sumber admin | A2 | 422 | Hanya draft penambahan langsung yang dapat diperbarui. | Tidak ada |
| Kirim pengajuan tanpa jenis asset / alasan | B2 | 422 | Jenis asset wajib diisi. / Alasan pengajuan wajib diisi. (induk: Data pengajuan asset belum lengkap.) | Tidak ada |
| Edit pengajuan yang bukan draft | B2 | 422 | Pengajuan asset sudah tidak dapat diubah karena bukan draft. | Tidak ada |
| Alokasi/tolak pengajuan yang sudah disetujui atau ditolak | B4 / B5 / tolak | 422 | Pengajuan asset sudah tidak dapat dialokasikan. / …tidak menunggu persetujuan. | Tidak ada |
| Tolak tanpa alasan | tolak | 422 | Alasan penolakan wajib diisi. | Tidak ada |
| Konfirmasi pengajuan yang belum disetujui admin | B6 | 422 | Pengajuan asset belum disubmit oleh admin. | Tidak ada |
| Konfirmasi dua kali | A3 / B6 | 422 | Penambahan asset sudah tidak menunggu konfirmasi. | Tidak ada |
| Tidak semua item dinilai | A3 / B6 | 422 | Seluruh asset penambahan wajib dicek. | Tidak ada |
| Rusak tanpa alasan | A3 / B6 | 422 | Alasan kerusakan wajib diisi. | Tidak ada |
| Pengguna bukan admin asset | A0–A2, B3–B5 | 403 | Pengguna tidak memiliki akses admin asset. | Tidak ada |
| Karyawan membuka transaksi orang lain / draft | A3 / B6 | 404 / 422 | Penambahan asset tidak ditemukan. | Tidak ada |
| Token kedaluwarsa | semua | 401 | Token telah expired. Mobile me-refresh token sekali lalu mengulang request. | Tidak ada |

Semua pelanggaran aturan dilempar sebagai `InvalidArgumentException` di dalam `DB::transaction`, sehingga seluruh perubahan request itu dibatalkan. Pesan tersedia dalam bahasa Indonesia dan Inggris mengikuti header `locale` (respons `en` memakai teks Inggris, mis. "Asset addition confirmed successfully.").

#### Ringkasan siapa mengerjakan apa

| Tahap | Mobile | API | Database |
|---|---|---|---|
| Opsi / pilih asset | Bottom sheet, state lokal | `getOptions` (SELECT) | Tidak berubah |
| Simpan / kirim penambahan | Rakit `AssetRequestKaryawanRequest` → `toJson` | Validasi, lock, cek asset & karyawan, ID, insert/update, reservasi hanya saat kirim | header, item, `asset_data → proses` (kirim), `notif_data` (kirim) |
| Simpan / kirim pengajuan | `AssetPengajuanKaryawanRequest` (`jenis_asset`, `keterangan`) | Validasi, ID, insert/update header tanpa item | header, `notif_data` (kirim) |
| Tolak / alokasi | Bottom sheet alasan / form setujui | Validasi status & asset, ganti item, tentukan approval | item, header, `asset_data → proses` (kirim alokasi), `notif_data` |
| Konfirmasi | `_statusByItem`, `_ketRusakByItem`; sama untuk kedua jalur | Hitung `semuaNormal`; serah terima hanya bila semua normal | item, header, `asset_data`, `asset_trx_log`, `notif_data` |
| Tampilan | JSON → model → widget; label dari `status_label` dan `tab` | `requestStatusMetadata` / `pengajuanStatusMetadata` / `pengajuanAdminStatusMetadata` | `SELECT` saja |

---

## 46. Ringkasan Database Impact

| Event | INSERT | UPDATE | DELETE |
|---|---|---|---|
| Buat Penambahan (draft) | header, item | – | – |
| Buat Penambahan (kirim) | header, item, notif | asset_data → proses | – |
| Buat Pengajuan | header, notif (kirim) | – | – |
| Tolak | notif | header (ditolak) | item (draft alokasi) |
| Alokasi draft | item | header (jadwal) | item lama |
| Alokasi kirim (approval) | item, notif | header (jadwal, approval), asset_data → proses | item lama |
| Konfirmasi semua normal | asset_trx_log, notif | item, header (dikonfirmasi), asset_data → terpakai/karyawan | – |
| Konfirmasi ada rusak | notif | item, header (tetap proses) | – |

---

## 47. Checklist Implementasi

### Database
- [x] Header & item
- [x] Kolom `sumber_pengajuan` + index list
- [x] Status enum
- [x] Log serah terima
- [ ] Migration CREATE TABLE baseline
- [ ] Unique `no_trx` per company
- [ ] Kolom tanggal nullable
- [x] SQL manual `sql-pengajuan-karyawan-jenis-asset-bebas.sql` + DBML: `id_asset_kategori_m` dan `urgensi` nullable (v1.4) disiapkan

### Backend
- [x] Route, controller, validasi, transaksi, notifikasi
- [x] Tolak & alokasi (termasuk draft alokasi)
- [x] Konfirmasi karyawan
- [x] Edit / kirim draft (`PUT`); hapus draft belum ada
- [ ] Tindak lanjut asset rusak
- [x] Perbaikan `hasActiveRequestAsset` (v1.1)
- [x] Pengajuan: `jenis_asset` teks bebas, urgensi dihapus, alokasi lintas kategori (v1.4; 99 tes API lulus)
- [ ] Validasi tanggal serah terima ≥ hari ini

### Mobile
- [x] List, form, detail Penambahan (admin)
- [x] List, detail, tolak, setujui Pengajuan (admin)
- [x] List, form, detail Pengajuan (karyawan)
- [x] Konfirmasi Penambahan/Pengajuan
- [x] Lanjutkan (ubah) draft
- [x] Form Jenis Asset teks bebas tanpa Urgensi; Bloc tanpa options (v1.4)
- [x] Data Karyawan menampilkan Jabatan, Cabang, Departemen
- [x] Tombol pil (Tambah Asset, Simpan Draft, Submit ke PIC, Tolak, Setujui); nilai awal jadwal/jam layar Setujui kosong (38 tes mobile lulus)
- [ ] Tampilkan kondisi item di detail admin
- [ ] Semua banner penambahan di Asset Saya

### Rilis / Deploy v1.4
- [ ] **Jalankan `sql-pengajuan-karyawan-jenis-asset-bebas.sql` di database SEBELUM deploy API v1.4** (langkah user; belum dijalankan; cek charset `nama_kategori` lebih dulu)
- [ ] Deploy API v1.4
- [ ] Rilis aplikasi mobile v1.4 (klien lama tanpa `jenis_asset` ditolak 422)

### Documentation
- [x] Business Flow, DB, API, Mobile, Mapping, Manual Book, Testing
- [ ] Screenshot

---

## 48. Changelog

| Version | Date | Changes |
|---|---|---|
| 1.5 | 2026-10-22 | Menambah §45.4 *Walkthrough Lengkap dengan Contoh Data* untuk Penambahan (admin) dan Pengajuan (karyawan): aksi mobile, request, proses API, isi database per langkah, notifikasi, variasi rusak/tolak, kasus gagal. Sinkronisasi kode: tabel log ternyata `asset_trx_log` (`id_asset_trx_log`), bukan `asset_log`; konfirmasi semua normal juga mengosongkan `asset_data.id_asset_lokasi_m`; pesan API kini dua bahasa mengikuti header `locale`. |
| 1.4 | 2026-10-20 | Pengajuan karyawan: **Jenis Asset teks bebas** (`jenis_asset`, maks 150) menggantikan pilihan kategori; **Urgensi dihapus** dari form, validasi, response, dan options. `id_asset_kategori_m`/`urgensi` dari klien lama diabaikan dan disimpan NULL; `nama_kategori` = teks jenis asset (BR-007, BR-008, API-010/012/017, §18, §21, §24, §26-§31). Persetujuan admin **tidak lagi dibatasi kategori** (BR-011, API-007/009). SQL manual `sql-pengajuan-karyawan-jenis-asset-bebas.sql` (kolom nullable) **belum dijalankan; jalankan sebelum deploy API**. Kartu Data Karyawan menampilkan Jabatan, Cabang, Departemen (`karyawan.jabatan`). Mobile: tombol bentuk pil, perbaikan nilai awal jadwal/jam layar Setujui. Kompatibilitas: submit klien lama tanpa `jenis_asset` -> 422; draft lama mempertahankan `nama_kategori`. Tes: API 99 lulus, mobile 38 lulus. Ditambah TC-39..TC-41, RISK-012/013, INC-12. |
| 1.3 | 2026-10-12 | Mobile: stepper alur status (`AssetAlurStepper.pengajuan`) di detail penambahan/pengajuan admin dan detail pengajuan karyawan (§26.3, 26.5, 26.9). Teks langkah di grup JSON `alur`. Tidak ada perubahan API. |
| 1.2 | 2026-10-12 | Sinkronisasi dengan kode: **edit/kirim draft** kini tersedia — `PUT /asset/penambahan-karyawan/{id}` dan `PUT /asset/karyawan/pengajuan/{id}` (API-016, API-017); mobile membuka draft untuk diubah (Ubah Draft). RISK-003 diturunkan menjadi RENDAH (hapus draft belum ada); jumlah endpoint menjadi 10 admin + 7 karyawan (17). Path mobile mengikuti struktur folder baru |
| 1.0 | 2026-09-26 | Initial documentation — disusun dari source lokal API (Lumen) & MOBILE (Flutter) |
| 1.1 | 2026-09-29 | RISK-002 diperbaiki: `hasActiveRequestAsset` hanya mengunci asset pada header `proses`; asset hasil retur bisa ditambahkan lagi. Update BR-005, TC-30/30a, Edge Case, Dependency, Checklist |
