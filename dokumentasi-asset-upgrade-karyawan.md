# Dokumentasi Teknis & Manual Book — Upgrade Asset Karyawan

> Dokumen ini disusun **murni dari source code lokal** (`API/` Lumen dan `MOBILE/` Flutter) per 30 Oktober 2026 (v1.4).
> Business requirement dan kode fitur **diturunkan dari kode**. Bagian yang tidak ditemukan ditandai **⚠️ Belum ditemukan**, asumsi ditandai **⚠️ Asumsi**.
> Contoh ID/nama pada payload adalah **data sintetis**.

---

## 1. Informasi Dokumen

| Informasi | Nilai |
|---|---|
| Nama Fitur | Upgrade Asset Karyawan — tipe **Upgrade Komponen** (UI: **"Upgrade Asset"**; menu **"Upgrade"**) |
| Kode Fitur | **KRY03** (nomor transaksi `KRY03-2-YYYYMMNNN`; docblock model: `ALK-KRY03 tipe Upgrade Komponen`) |
| Modul | Asset Management |
| Backend | Laravel **Lumen ^10.0**, PHP ^8.1, JWT, Query Builder |
| Database | `API/diagram-asset.dbml` + migration `2026_09_26_000001_add_workflow_fields_to_asset_upgrade_karyawan.php`. ⚠️ Asumsi: MySQL/MariaDB |
| Mobile | Flutter (`professional_flutter_3_35`), `flutter_bloc`, `dio` |
| Web | ⚠️ Belum ditemukan: tidak ada source web di workspace |
| Test otomatis | `API/tests/Asset/AssetUpgradeKaryawanTest.php` (5 test, 54 assertion) · `MOBILE/test/asset/asset_upgrade_test.dart` (5 test) |
| Versi Dokumen | 1.4 |
| Tanggal Dokumentasi | 2026-10-30 |
| Skrip data lama | `Claude outputs/sql-upgrade-hapus-checklist-review-lama.sql` — menyelesaikan transaksi lama di fase `menunggu_review` (**BELUM dijalankan**) |
| Dokumen terkait | Onboarding (KRY01), Offboarding (KRY02), Retur (KRY04), Peminjaman (KRY05), Kerusakan (KRY06), Request/Penambahan |

---

## 2. Daftar Isi

1. Informasi Dokumen · 3. Quick Overview · 4. Ringkasan Fitur · 5. Tujuan · 6. Aktor · 7. Hak Akses · 8. Terminologi · 9. Business Rules · 10. Business Flow · 11. Flow per Role · 12. Flow Diagram · 13. Sequence Diagram · 14. Status Transaksi · 15. State Transition · 16. Struktur Database · 17. Relasi · 18. Data Dictionary · 19. Index · 20. Backend Architecture · 21. Backend Logic · 22. Dampak asset_data · 23. API · 24. Detail API · 25. Mobile Architecture · 26. Mobile Screen · 27. Mobile → API · 28. API → Mobile · 29. Mapping Mobile→API→DB · 30. Mapping DB→API→Mobile · 31. Validation · 32. Error Handling · 33. Audit Trail · 34. Notification · 35. Security · 36. Manual Book · 37. Kondisi Button · 38. Testing · 39. Temuan Ketidaksesuaian · 40. Edge Case · 41. Risk · 42. Dependency · 43. Source Reference · 44. Developer Notes · 45. Traceability · 46. DB Impact · 47. Checklist · 48. Changelog

---

## 3. Quick Overview

| Item | Informasi |
|---|---|
| Feature | Upgrade spesifikasi komponen **satu asset** milik karyawan (mis. RAM 8 GB → 16 GB). Asset diambil sementara, diupgrade di luar sistem, lalu dikembalikan ke **karyawan yang sama** |
| Module | Asset Management |
| Jalur | **A — Karyawan mengajukan** (perlu persetujuan admin) · **B — Admin menginisiasi** (langsung disetujui + jadwal) |
| Actor | Karyawan, Admin Asset, System |
| Main Transaction | `asset_trx_upgrade_karyawan` |
| Detail Transaction | `asset_trx_upgrade_karyawan_item` — satu baris per **atribut spesifikasi** (field `komponen` di `asset_data_item`) |
| Asset Master | `asset_data` (status `dalam_perbaikan` selama service) + `asset_data_item` (nilai atribut diperbarui saat karyawan menekan **Terima**) |
| Log | `asset_trx_log` `jenis_log = upgrade_karyawan` |
| Status | `status_approval` (proses/disetujui/ditolak) + `fase_upgrade` (`pengajuan → menunggu_pengambilan → dalam_service → menunggu_pengembalian → selesai`). Fase `menunggu_review` tidak dihasilkan lagi sejak v1.4 (hanya data lama) |
| Hasil | `selesai` (nilai baru ditulis ke atribut) atau `tidak_dapat_diupgrade` (atribut tidak berubah) + biaya |
| Serah terima | **Tanpa checklist** sejak v1.4. Karyawan menekan **Serahkan Aset** (pelepasan) dan **Terima Aset**; **Terima langsung menyelesaikan** transaksi. Tidak ada review akhir dan tidak ada kirim ulang |
| API | 7 endpoint admin + 6 endpoint karyawan = **13 endpoint** (sumber: `RouteAsset.php`; berkurang 2 dari v1.3) |
| Mobile Screen | Upgrade Asset (list), Ajukan / Buat Upgrade Asset (form), Detail Upgrade Asset — dipakai bersama admin & karyawan |

```text
Jalur A: Karyawan pilih asset + atribut + detail upgrade + alasan + urgensi → Ajukan Upgrade
         Admin: Tolak (+alasan) ─────────────────────────────────── ditolak / selesai
                Setujui & Jadwalkan (jadwal pengambilan) ─┐
Jalur B: Admin pilih karyawan + asset + atribut + catatan + jadwal → Buat Upgrade ─┤
                                                          menunggu_pengambilan ◄────┘
Karyawan: Serahkan Aset (tanpa checklist) ─────── dalam_service · asset = dalam_perbaikan (pemegang tetap karyawan)
Admin: Hasil Upgrade (Selesai / Tidak Dapat Diupgrade, nilai baru, biaya, keterangan) + jadwal pengembalian
                                                          menunggu_pengembalian
Karyawan: Terima Aset (tanpa checklist) ─────────────── selesai
          → atribut diperbarui (bila berhasil), asset = terpakai, log gudang → karyawan
```

---

## 4. Ringkasan Fitur

Upgrade Asset dipakai untuk meningkatkan spesifikasi komponen asset yang dipegang karyawan tanpa mengganti unitnya. Yang diupgrade adalah **atribut spesifikasi** (misalnya "RAM", "Storage") yang tersimpan sebagai field `komponen` di `asset_data_item` (unit utama maupun kelengkapan).

Ada dua jalur masuk: karyawan mengajukan (dengan alasan dan urgensi) lalu admin menyetujui atau menolak, atau admin langsung membuat upgrade untuk karyawan tertentu. Setelah disetujui dan dijadwalkan, karyawan menekan **Serahkan Aset**, teknisi mengerjakan di luar sistem, admin mencatat hasil beserta jadwal pengembalian, lalu karyawan menekan **Terima Aset**. Sejak v1.4 tidak ada lagi checklist kondisi (pengambilan maupun penerimaan), tidak ada review akhir admin, dan tidak ada kirim ulang ke teknisi: tombol **Terima Aset** langsung menyelesaikan transaksi.

Selama proses, **pemegang asset tetap karyawan**; hanya `status_asset` yang berubah menjadi `dalam_perbaikan`.

---

## 5. Tujuan Fitur

### 5.1 Tujuan Bisnis (diturunkan dari kode)
- Karyawan dapat meminta peningkatan spesifikasi asset kerja.
- Admin dapat menjadwalkan upgrade secara proaktif.
- Biaya dan hasil upgrade terdokumentasi.
- Data spesifikasi asset otomatis terbarui setelah upgrade berhasil dan karyawan menerima asset kembali.
- Sejak v1.4 kondisi fisik komponen sebelum dan sesudah upgrade **tidak lagi dicatat** di sistem (checklist dihapus).

### 5.2 Tujuan Sistem
- Menyimpan atribut target beserta nilai lama/baru per item.
- Menandai asset `dalam_perbaikan` selama service dan mengembalikannya ke `terpakai`.
- Menulis nilai baru ke `asset_data_item.json_value`.
- Mencegah asset diproses di upgrade lain, kerusakan, retur, atau offboarding saat upgrade dibuat.

---

## 6. Aktor

| Aktor | Role | Fungsi |
|---|---|---|
| Karyawan | User login aktif, pemegang asset | Mengajukan (Jalur A), menyerahkan asset, menerima asset hasil upgrade (sekaligus menyelesaikan transaksi) |
| Admin Asset | `asset_admin` + karyawan aktif | Review, membuat upgrade (Jalur B), jadwal, input hasil |
| Teknisi | Di luar sistem | ⚠️ Belum ditemukan: tidak ada entitas/aktor teknisi di kode |
| System | Backend + FCM | Validasi, lock, penomoran, update asset & atribut, log, notifikasi |

---

## 7. Hak Akses

| Action | Admin Asset | Karyawan | Keterangan |
|---|:---:|:---:|---|
| Melihat semua upgrade | ✅ | ❌ | `/asset/upgrade-karyawan` |
| Melihat upgrade sendiri | – | ✅ | Tanpa cek karyawan aktif |
| Mengajukan upgrade (Jalur A) | – | ✅ | Karyawan aktif, asset miliknya `terpakai` |
| Membuat upgrade untuk karyawan (Jalur B) | ✅ | ❌ | Langsung disetujui |
| Setujui / tolak pengajuan | ✅ | ❌ | `status_approval = proses` |
| Ubah jadwal | ✅ | ❌ | `menunggu_pengambilan` / `menunggu_pengembalian` |
| Serahkan aset (pelepasan) | ❌ | ✅ | `menunggu_pengambilan`; **tanpa checklist** (v1.4); hanya pemilik transaksi |
| Input hasil upgrade | ✅ | ❌ | `dalam_service` |
| Terima aset (langsung selesai) | ❌ | ✅ | `menunggu_pengembalian`; **tanpa checklist/body** (v1.4); hanya pemilik transaksi |
| Checklist pengambilan / penerimaan | – | – | **Dihapus v1.4** |
| Selesaikan / kirim ulang (review akhir) | – | – | **Dihapus v1.4** |
| Batalkan upgrade | ❌ | ❌ | ⚠️ Belum ditemukan (RISK-003) |

Karyawan lain maupun PIC tidak dapat menyerahkan atau menerima upgrade milik orang lain: `lockHeader` memfilter `id_karyawan`, sehingga transaksi orang lain dijawab "Upgrade asset tidak ditemukan." (422).

---

## 8. Terminologi

| Istilah | Arti di sistem |
|---|---|
| Atribut | Satu field `komponen` (bukan tipe `file`) di `asset_data_item.json_value` dengan `flag_spesifikasi` 1/2; ID = `id_asset_data_item|id_komponen` |
| Detail upgrade | Target spesifikasi yang diminta (mis. "8 GB menjadi 16 GB") |
| Nilai lama / baru | `nilai_lama` = nilai atribut saat pengajuan; `nilai_baru` = hasil yang diisi admin |
| Jalur A / B | `sumber_pengajuan = karyawan` / `admin` |
| Urgensi | `normal`, `secepatnya`, `mendesak` (Jalur A) |
| Pelepasan | Karyawan menyerahkan asset untuk diupgrade (tombol **Serahkan Aset**, tanpa checklist) |
| Terima | Karyawan menerima asset hasil upgrade (tombol **Terima Aset**, tanpa checklist); langsung menyelesaikan transaksi |
| Checklist pengambilan / penerimaan | **Dihapus v1.4**. Kolom `checklist_pengambilan` dan `checklist_pengembalian` tetap ada di tabel tetapi tidak dipakai |
| Review akhir / kirim ulang | **Dihapus v1.4**. Fase `menunggu_review` hanya tersisa pada data lama |

---

## 9. Business Rules

### BR-001 — Asset yang boleh diupgrade
Dipegang karyawan (`pemegang_asset = karyawan`, `id_pemegang_asset` = karyawan transaksi), `status_asset = terpakai`, dan memiliki minimal satu atribut spesifikasi → "Asset bukan milik karyawan atau tidak ditemukan." / "Hanya asset berstatus terpakai yang dapat diupgrade." Karyawan harus aktif → "Karyawan tidak ditemukan atau tidak aktif."

### BR-002 — Asset tidak sedang di flow lain
Saat opsi dan simpan: upgrade aktif, laporan kerusakan aktif, retur aktif, offboarding aktif, pemeliharaan terjadwal (PML01) aktif → "Asset sedang diproses pada upgrade lain." / "…laporan kerusakan." / "…retur." / "…offboarding." / "…pemeliharaan."

### BR-003 — Item atribut
Minimal 1; atribut harus milik asset; tanpa duplikat; detail upgrade wajib (≤255) → "Minimal satu komponen wajib dipilih." / "Komponen upgrade tidak ditemukan pada asset." / "Komponen yang sama tidak boleh dipilih lebih dari sekali." / "Detail upgrade setiap komponen wajib diisi." Snapshot atribut disimpan di `kelengkapan_item` (JSON) dan `nilai_lama`.

### BR-004 — Jalur A (karyawan)
Alasan wajib (≤1000), urgensi wajib (normal/secepatnya/mendesak). Status awal `proses` / `pengajuan`.

### BR-005 — Jalur B (admin)
Karyawan dipilih dari opsi (aktif & memegang asset yang dapat diupgrade), catatan opsional (≤1000), **jadwal pengambilan wajib** dengan tanggal ≥ hari ini. Status awal `disetujui` / `menunggu_pengambilan`, approval diisi admin pembuat. Notifikasi `upgrade.disetujui` ke karyawan.

### BR-006 — Review (Jalur A)
Hanya saat `status_approval = proses` → "Pengajuan upgrade sudah direview."

- **Tolak**: alasan wajib → `ditolak`, `status_transaksi = selesai`, `fase = selesai`.
- **Setujui**: asset masih `terpakai` di karyawan; jadwal pengambilan wajib (≥ tanggal pengajuan) → `disetujui`, `menunggu_pengambilan`.

### BR-007 — Jadwal
Format `Y-m-d` + `H:i`; tanggal ≥ tanggal pengajuan (Jalur B: ≥ hari ini) → "Tanggal dan jam serah terima wajib diisi." / "Format jadwal serah terima tidak valid." / "Tanggal serah terima tidak boleh sebelum tanggal pengajuan." Ubah jadwal hanya pada `menunggu_pengambilan` (jadwal pengambilan) atau `menunggu_pengembalian` (jadwal pengembalian) → "Jadwal hanya dapat diubah sebelum serah terima dilakukan." Kolom `keterangan_pengembalian` dipakai untuk catatan kedua jadwal.

### BR-008 — *(Dihapus v1.4)* Checklist pengambilan oleh admin
Aturan lama (admin wajib mengisi checklist semua komponen sebelum karyawan boleh menyerahkan) sudah dihapus. Endpoint `POST /asset/upgrade-karyawan/{id}/checklist-pengambilan` (API-007) tidak ada lagi dan karyawan dapat menyerahkan asset begitu fase `menunggu_pengambilan`. Nomor BR-008 dipertahankan agar rujukan lama tidak bergeser.

### BR-009 — Serahkan aset / pelepasan (karyawan)
Hanya pada fase `menunggu_pengambilan` → "Aset belum siap diserahkan untuk upgrade." **Tanpa syarat checklist** dan tanpa body request.

- Karyawan harus aktif dan pemilik transaksi (`lockHeader` memfilter `id_karyawan`; transaksi orang lain → "Upgrade asset tidak ditemukan.").
- Asset harus masih dipegang karyawan transaksi (`terpakai` atau `dalam_perbaikan`) → "Status asset sudah berubah atau tidak lagi dipegang karyawan."
- Asset → `status_asset = dalam_perbaikan` (**pemegang tetap karyawan**), log `upgrade_karyawan` posisi karyawan → gudang. Fase → `dalam_service`, `tgl_pelepasan` dan `user_pelepasan` diisi.

### BR-010 — Hasil upgrade (admin)
Hanya `dalam_service` → "Aset belum berada dalam proses upgrade."

- `status_hasil`: `selesai` / `tidak_dapat_diupgrade`; keterangan hasil wajib (≤1000); biaya ≥ 0.
- `selesai`: nilai baru setiap atribut wajib → "Nilai baru setiap atribut wajib diisi."
- `tidak_dapat_diupgrade`: `nilai_baru` dikosongkan, `alasan_tidak_upgrade` = keterangan (≤255).
- Jadwal pengembalian wajib. Fase → `menunggu_pengembalian`; `tgl_diterima` direset. (Sejak v1.4 tidak ada lagi checklist penerimaan yang direset.)

### BR-011 — Terima aset (karyawan) — langsung menyelesaikan
Hanya `menunggu_pengembalian` → "Aset belum dijadwalkan untuk dikembalikan." **Tanpa checklist dan tanpa body.** Karyawan harus aktif dan pemilik transaksi; asset harus `dalam_perbaikan` di karyawan → "Status asset sudah berubah atau tidak lagi dipegang karyawan."

- Bila `status_hasil = selesai`: nilai baru setiap item ditulis ke `asset_data_item.json_value` (`applyAtribut`). Bila `tidak_dapat_diupgrade`: atribut tidak berubah.
- Asset → `terpakai`; log `upgrade_karyawan` gudang → karyawan.
- Header: `fase_upgrade = selesai`, `status_transaksi = selesai`, `tgl_diterima = now`.
- Memanggil Terima sekali lagi (fase sudah `selesai`) ditolak dengan pesan yang sama: "Aset belum dijadwalkan untuk dikembalikan."

### BR-012 — *(Dihapus v1.4)* Review akhir admin (Selesaikan / Kirim Ulang)
Aksi admin **Selesaikan Upgrade** dan **Kirim Ulang ke Teknisi** beserta endpoint `POST /asset/upgrade-karyawan/{id}/selesai` (API-009) sudah dihapus; penyelesaian terjadi otomatis pada BR-011. Siklus ulang kirim ulang tidak ada lagi. Fase `menunggu_review` tidak dihasilkan lagi (konstanta dan label dipertahankan hanya untuk data lama, lihat BR-015).

### BR-013 — Penomoran
Header & item `id_company + YYYYMM + 3 digit`; `no_trx = KRY03-2-YYYYMMNNN`; log 5 digit.

### BR-014 — Kondisi saat serah terima tidak tercatat (v1.4)
Karena checklist dihapus, `asset_trx_log` selalu menulis `kondisi_akhir = kondisi_awal` (nilai `asset_data.kondisi_awal`). Kerusakan yang ditemukan saat pengambilan atau penerimaan harus dilaporkan lewat alur Kerusakan (KRY06), bukan lewat Upgrade.

### BR-015 — Data lama pada fase `menunggu_review` (v1.4)
Transaksi yang sudah berhenti di `menunggu_review` sebelum v1.4 **tidak memiliki aksi lagi** (`aksi` semuanya `false`) dan asetnya tetap `dalam_perbaikan` (terkunci). Penyelesaiannya satu kali lewat skrip `Claude outputs/sql-upgrade-hapus-checklist-review-lama.sql` (**belum dijalankan**). Lihat RISK-008.

---

## 10. Business Flow

```text
Jalur A
1. Karyawan: Asset Saya → Upgrade → (+) Ajukan Upgrade Asset
   → Pilih Asset → pilih atribut + Detail Upgrade → Alasan → Urgensi → Ajukan Upgrade   [Menunggu Persetujuan · notif admin]
2. Admin: Detail → Tolak (+Alasan Penolakan)                                             [Ditolak · notif karyawan]
          atau Setujui & Jadwalkan (Jadwal Pengambilan)                                  [Menunggu Pengambilan · notif karyawan]
Jalur B
1. Admin: Upgrade → (+) Buat Upgrade Asset → Pilih Karyawan → Pilih Asset → atribut + Detail Upgrade
   → Catatan → Jadwal Pengambilan → Buat Upgrade                                         [Menunggu Pengambilan · notif karyawan]
Lanjutan (kedua jalur)
3. Karyawan: Serahkan Aset (tanpa checklist)                                             [Dalam Service · notif admin]
4. Admin: Hasil Upgrade (Upgrade Selesai / Tidak Dapat Diupgrade, Nilai Baru, Keterangan, Biaya)
   → Simpan Hasil & Jadwalkan (Jadwal Pengembalian)                                      [Menunggu Pengembalian · notif karyawan]
5. Karyawan: Terima Aset (tanpa checklist) → langsung selesai,
   atribut diperbarui bila berhasil, asset terpakai                                      [Selesai · notif admin]
Admin dapat Ubah Jadwal selama Menunggu Pengambilan dan Menunggu Pengembalian.
```

---

## 11. Flow Berdasarkan Role

### 11.1 Karyawan
1. **Asset Saya** → menu **Upgrade** → **Upgrade Asset** → **Ajukan Upgrade**.
2. Banner Asset Saya: **Serahkan Aset untuk Upgrade** ("{asset} siap diambil untuk diupgrade. Konfirmasi setelah diserahkan.") dan **Terima Aset Hasil Upgrade** ("{asset} selesai diupgrade. Cek kondisinya lalu terima."). Banner pelepasan muncul begitu fase `menunggu_pengambilan` (tanpa menunggu checklist).
3. Detail → **Serahkan Aset** (`aksi.pelepasan`) atau **Terima Aset** (`aksi.terima`). Keduanya hanya dialog konfirmasi, tanpa form checklist.

### 11.2 Admin Asset
1. **Asset Karyawan** → aksi **Upgrade** atau panel **Upgrade Perlu Diproses** ("{jumlah} upgrade menunggu tindakan Admin/HR."): review pengajuan dan input hasil.
2. **Buat Upgrade** untuk Jalur B.
3. Detail → **Setujui & Jadwalkan** / **Tolak**, **Simpan Hasil & Jadwalkan**, **Ubah Jadwal**. Admin tidak lagi mengisi checklist dan tidak ada tombol Selesaikan / Kirim Ulang.

### 11.3 System
Deteksi atribut, cek flow lain, update status asset, tulis nilai atribut (saat Terima), log, notifikasi.

---

## 12. Flow Diagram

```mermaid
flowchart TD
    A1[Jalur A: Karyawan ajukan upgrade] --> R{Admin review}
    R -->|Tolak| T[Ditolak]
    R -->|Setujui + jadwal| P[Menunggu Pengambilan]
    B1[Jalur B: Admin buat upgrade + jadwal] --> P
    P --> S[Karyawan: Serahkan Aset, tanpa checklist]
    S --> D[Dalam Service, asset dalam_perbaikan]
    D --> H{Admin: Hasil Upgrade}
    H -->|Upgrade Selesai + nilai baru| K[Menunggu Pengembalian + jadwal]
    H -->|Tidak Dapat Diupgrade| K
    K --> G[Karyawan: Terima Aset, tanpa checklist]
    G --> F[Selesai: atribut diperbarui bila berhasil, asset terpakai]
```

---

## 13. Sequence Diagram

```mermaid
sequenceDiagram
    actor K as Karyawan
    actor A as Admin
    participant M as Mobile
    participant CK as AssetKaryawanUpgrade_c
    participant CA as AssetAdminUpgrade_c
    participant U as AssetUpgradeKaryawan_m
    participant DB
    participant N as AssetNotification

    K->>M: Pilih asset, atribut, detail, alasan, urgensi
    M->>CK: POST /asset/karyawan/upgrade
    CK->>U: storeKaryawan()
    U->>DB: cek karyawan, asset terpakai, flow lain, INSERT header + item
    CK->>N: upgrade.diajukan (admin)
    A->>M: Setujui & Jadwalkan
    M->>CA: POST /asset/upgrade-karyawan/{id}/review
    CA->>U: review()
    U->>DB: UPDATE header disetujui, menunggu_pengambilan + jadwal
    CA->>N: upgrade.disetujui (karyawan)
    K->>M: Serahkan Aset (tanpa checklist)
    M->>CK: POST /asset/karyawan/upgrade/{id}/pelepasan
    CK->>U: pelepasan()
    U->>DB: asset dalam_perbaikan, INSERT asset_trx_log, fase dalam_service
    CK->>N: upgrade.dilepas (admin)
    A->>M: Hasil upgrade, nilai baru, biaya, jadwal pengembalian
    M->>CA: POST /asset/upgrade-karyawan/{id}/hasil
    CA->>U: hasil()
    U->>DB: UPDATE item nilai_baru, header menunggu_pengembalian
    CA->>N: upgrade.pengembalian (karyawan)
    K->>M: Terima Aset (tanpa checklist)
    M->>CK: POST /asset/karyawan/upgrade/{id}/terima
    CK->>U: terima()
    U->>DB: tulis nilai baru ke asset_data_item (bila berhasil), asset terpakai, INSERT asset_trx_log, fase selesai
    CK->>N: upgrade.diterima (admin, "Upgrade selesai.")
```

---

## 14. Status Transaksi

Tiga kolom: `status_approval` (enum `proses/disetujui/ditolak`), `fase_upgrade` (enum, migration), `status_transaksi` (enum `proses/selesai`).

| status_approval | fase_upgrade | Label API | Tab | Aksi berikutnya |
|---|---|---|---|---|
| proses | pengajuan | Menunggu Persetujuan | pengajuan | Admin: setujui / tolak |
| ditolak | selesai | Ditolak | selesai | – |
| disetujui | menunggu_pengambilan | Menunggu Pengambilan | proses | Karyawan: serahkan aset; Admin: ubah jadwal |
| disetujui | dalam_service | Dalam Service | proses | Admin: input hasil |
| disetujui | menunggu_pengembalian | Menunggu Pengembalian | proses | Karyawan: terima (langsung selesai); Admin: ubah jadwal |
| disetujui | menunggu_review | Menunggu Review | proses | **Hanya data lama (sebelum v1.4)**: tidak ada aksi; diselesaikan skrip SQL (BR-015) |
| disetujui | selesai | Selesai | selesai | – |

Detail API mengembalikan objek `aksi` yang kini hanya berisi: `review`, `ubah_jadwal`, `pelepasan`, `hasil`, `terima`. Kunci `checklist_pengambilan` dan `selesai` dihapus (juga field detail `checklist_pengambilan`, `checklist_pengembalian`, `checklist_pengembalian_rusak`).

---

## 15. State Transition

```mermaid
stateDiagram-v2
    [*] --> pengajuan: Ajukan (Karyawan, Jalur A)
    [*] --> menunggu_pengambilan: Buat upgrade (Admin, Jalur B)
    pengajuan --> ditolak: Tolak (Admin)
    pengajuan --> menunggu_pengambilan: Setujui + jadwal (Admin)
    menunggu_pengambilan --> menunggu_pengambilan: Ubah jadwal (Admin)
    menunggu_pengambilan --> dalam_service: Serahkan aset (Karyawan)
    dalam_service --> menunggu_pengembalian: Input hasil + jadwal (Admin)
    menunggu_pengembalian --> menunggu_pengembalian: Ubah jadwal (Admin)
    menunggu_pengembalian --> selesai: Terima aset (Karyawan)
    ditolak --> [*]
    selesai --> [*]
```

Fase `menunggu_review` tidak ada pada diagram karena tidak dihasilkan lagi sejak v1.4; transaksi lama di fase itu hanya bisa diselesaikan lewat skrip SQL.

Status `asset_data`:

| Fase | status_asset | pemegang |
|---|---|---|
| pengajuan / menunggu_pengambilan | terpakai | karyawan |
| dalam_service / menunggu_pengembalian | dalam_perbaikan | karyawan (tidak berubah) |
| selesai | terpakai | karyawan |
| menunggu_review (data lama) | dalam_perbaikan (terkunci) | karyawan |

---

## 16. Struktur Database

| Table | Type | Fungsi |
|---|---|---|
| `asset_trx_upgrade_karyawan` | Transaction Header | Pengajuan, jadwal, fase, hasil, biaya (kolom `checklist_pengambilan` / `checklist_pengembalian` **tidak dipakai sejak v1.4**) |
| `asset_trx_upgrade_karyawan_item` | Transaction Detail | Atribut target, detail, nilai lama/baru |
| `asset_data` | Master | Status asset |
| `asset_data_item` | Master komponen | Nilai atribut (`json_value.komponen[].value`) — **diperbarui** saat karyawan menekan Terima |
| `asset_kategori_m` | Master | Snapshot kategori |
| `asset_trx_log` | Log | `upgrade_karyawan` (enum ditambah oleh migration) |
| `data_karyawan`, `master_cabang`, `master_departemen` | Master HR | |
| `asset_admin`, `master_company` | Akses / lock | |
| `notif_data`, `users` | Notifikasi | |

> ⚠️ Belum ditemukan: migration `CREATE TABLE` untuk kedua tabel; migration `2026_09_26_000001` hanya menambah kolom workflow, index, dan tipe log.

---

## 17. Relasi Database

```mermaid
erDiagram
    ASSET_TRX_UPGRADE_KARYAWAN ||--o{ ASSET_TRX_UPGRADE_KARYAWAN_ITEM : "id_asset_trx_upgrade"
    ASSET_DATA ||--o{ ASSET_TRX_UPGRADE_KARYAWAN_ITEM : "id_asset_data"
    ASSET_DATA ||--o{ ASSET_DATA_ITEM : "id_asset_data"
    ASSET_DATA_ITEM ||--o{ ASSET_TRX_UPGRADE_KARYAWAN_ITEM : "kelengkapan_item.id_asset_data_item (JSON)"
    DATA_KARYAWAN ||--o{ ASSET_TRX_UPGRADE_KARYAWAN : "id_karyawan (logis)"
    ASSET_DATA ||--o{ ASSET_TRX_LOG : "id_asset_data"

    ASSET_TRX_UPGRADE_KARYAWAN {
        varchar id_asset_trx_upgrade_karyawan PK
        varchar no_trx
        enum sumber_pengajuan
        varchar urgensi
        enum status_approval
        enum fase_upgrade "menunggu_review hanya data lama"
        enum status_transaksi
        text checklist_pengambilan "tidak dipakai sejak v1.4"
        text checklist_pengembalian "tidak dipakai sejak v1.4"
        enum status_hasil
        bigint biaya_upgrade
    }
    ASSET_TRX_UPGRADE_KARYAWAN_ITEM {
        varchar id_asset_trx_upgrade_item PK
        varchar id_asset_trx_upgrade FK
        varchar id_asset_data FK
        text kelengkapan_item
        varchar detail_upgrade
        varchar nilai_lama
        varchar nilai_baru
        varchar alasan_tidak_upgrade
    }
    ASSET_DATA {
        varchar id_asset_data PK
        enum status_asset
        enum pemegang_asset
    }
    ASSET_DATA_ITEM {
        varchar id_asset_data_item PK
        varchar id_asset_data FK
        text json_value
    }
    ASSET_TRX_LOG {
        varchar id_asset_trx_log PK
        varchar id_asset_data FK
        enum jenis_log
    }
```

---

## 18. Data Dictionary

### 18.1 asset_trx_upgrade_karyawan
| Field | Type | Default | Description |
|---|---|---|---|
| id_asset_trx_upgrade_karyawan | varchar(50) PK | – | `id_company + YYYYMM + 3 digit` |
| no_trx | varchar(50) | '' | `KRY03-2-YYYYMMNNN` (DBML: unique) |
| sumber_pengajuan | enum(karyawan, admin) | karyawan | Jalur A / B (migration) |
| id_asset_kategori_m, nama_kategori | varchar | '' | Snapshot kategori |
| detail_upgrade | varchar(255) | '' | Ringkasan "label: detail; …" (dipotong 255) |
| alasan | text | – | Jalur A |
| urgensi | varchar(50) | – | Jalur A: normal/secepatnya/mendesak; Jalur B: '' |
| catatan | text | – | Jalur B |
| id_karyawan, nik_karyawan, nama_karyawan | varchar | '' | Snapshot karyawan |
| tgl_pengajuan | date | – | |
| tgl_pengambilan / jam_pengambilan | date / time | 0000-00-00 / 00:00:00 | Jadwal pengambilan |
| tgl_pengembalian / jam_pengembalian | date / time | 0000-00-00 / 00:00:00 | Jadwal pengembalian |
| keterangan_pengembalian | varchar(255) | '' | Catatan jadwal (dipakai untuk kedua jadwal) |
| checklist_pengambilan | text (JSON) | NULL | **Tidak dipakai sejak v1.4**: tidak ditulis dan tidak dibaca, kolom tidak diubah/dihapus. Data lama berformat `[{id_item, nama_item, jenis, kondisi, ket_rusak}]` |
| checklist_pengembalian | text (JSON) | NULL | **Tidak dipakai sejak v1.4** (format data lama sama dengan checklist_pengambilan) |
| tgl_pelepasan / user_pelepasan | timestamp / varchar | NULL / '' | Serahkan aset (terakhir) |
| status_hasil | enum(selesai, tidak_dapat_diupgrade) | NULL | Hasil upgrade |
| keterangan_hasil | text | NULL | Wajib saat input hasil |
| biaya_upgrade | bigint | 0 | Rupiah |
| tgl_diterima | timestamp | NULL | Karyawan menekan Terima (sekaligus waktu selesai); direset saat admin input hasil |
| status_transaksi | enum(proses, selesai) | proses | |
| status_approval | enum(proses, disetujui, ditolak) | – | |
| fase_upgrade | enum(pengajuan, menunggu_pengambilan, dalam_service, menunggu_pengembalian, menunggu_review, selesai) | pengajuan | (migration). `menunggu_review` tidak dihasilkan lagi sejak v1.4 (hanya data lama) |
| ket_tolak, user_approval, tgl_approval | – | – | Keputusan admin |
| id_company, tgl_input, user_input, tgl_update, user_update | – | – | Audit |

### 18.2 asset_trx_upgrade_karyawan_item
| Field | Type | Default | Description |
|---|---|---|---|
| id_asset_trx_upgrade_item | varchar(50) PK | – | `id_company + YYYYMM + 3 digit` |
| id_asset_trx_upgrade | varchar(50) | – | → header |
| id_asset_data | varchar(50) | – | Asset (sama untuk semua item) |
| kelengkapan_item | text (JSON) | '' | Snapshot atribut `{id_atribut, id_asset_data_item, id_komponen, nama_item, jenis, label, satuan, nilai}` |
| nama_asset, deskripsi_asset, kategori_asset | varchar(255) | '' | Snapshot |
| detail_upgrade | varchar(255) | '' | Target spesifikasi |
| nilai_lama | varchar(255) | '' | Nilai atribut saat pengajuan (migration) |
| nilai_baru | varchar(255) | '' | Nilai hasil upgrade (migration) |
| alasan_tidak_upgrade | varchar(255) | '' | Keterangan bila tidak dapat diupgrade |
| id_company, tgl_input, user_input | – | – | |

### 18.3 asset_trx_log (`jenis_log = upgrade_karyawan`)
| Kejadian | posisi | pemegang di log | kondisi |
|---|---|---|---|
| Serahkan aset | karyawan → gudang | `pemegang_asset = gudang`, id/nama pemegang = karyawan | kondisi_awal → kondisi_awal |
| Terima aset (langsung selesai) | gudang → karyawan | karyawan | kondisi_awal → kondisi_awal |

Sejak v1.4 `kondisi_akhir` selalu sama dengan `kondisi_awal` (nilai `asset_data.kondisi_awal`): kondisi saat serah terima tidak lagi tercatat karena checklist dihapus.

---

## 19. Database Index

| Index | Table | Columns | Tujuan |
|---|---|---|---|
| `atuk_admin_list_idx` | header | (id_company, status_approval, fase_upgrade, tgl_input, id_asset_trx_upgrade_karyawan) | List admin |
| `atuk_employee_list_idx` | header | (id_company, id_karyawan, status_approval, fase_upgrade, tgl_input, id_asset_trx_upgrade_karyawan) | List karyawan |
| `atuki_upgrade_idx` | item | (id_asset_trx_upgrade, id_company, id_asset_data) | Item per header, cek asset aktif |

---

## 20. Backend Architecture

```text
RouteAsset.php (prefix /asset, jwt.api + jwt.ctx)
   ↓
AssetAdminUpgrade_c (validateRequest → mutateAssetTrx 'id_upgrade' / storeUpgradeKaryawan)
AssetKaryawanUpgrade_c (validateRequest → storeUpgradeSaya / mutateTrxSaya 'id_upgrade')
   ↓
AssetUpgradeKaryawan_m ← Asset_m (lockCompany, nextSequence, komponenAssets, parseJadwalValue,
   ↓                               insertAssetLog, findActiveKaryawan, findAssetAdmin)
   ↓   ↔ AssetRusakKaryawan_m / AssetReturKaryawan_m / AssetOffboardingKaryawan_m (cek flow lain)
DB::transaction (retry 3) → ApiResponse + AssetNotification
```

| Layer | File/Class | Function |
|---|---|---|
| Controller | `AssetAdminUpgrade_c` | getUpgradeOptions, getUpgradeKaryawan, getDetailUpgradeKaryawan, storeUpgradeKaryawan, reviewUpgradeKaryawan, updateJadwalUpgradeKaryawan, hasilUpgradeKaryawan (checklistPengambilanUpgrade dan selesaiUpgradeKaryawan dihapus v1.4) |
| Controller | `AssetKaryawanUpgrade_c` | getUpgradeOptionsSaya, getUpgradeSaya, storeUpgradeSaya, getDetailUpgradeSaya, pelepasanUpgradeSaya, terimaUpgradeSaya |
| Model | `AssetUpgradeKaryawan_m` | getOptions, getAdminOptions, getPage, getDetail, storeKaryawan, storeAdmin, review, updateJadwal, pelepasan, hasil, terima, getPendingForEmployee, getPendingForAdmin, countPendingForAdmin, hasActiveUpgradeAsset + helper |
| Support | `AssetNotification` | `upgrade.*` |

---

## 21. Backend Business Logic

### 21.1 Ajukan (Jalur A) — `storeKaryawan`
Lock → karyawan aktif → urgensi & alasan → `insertTransaction` (asset milik karyawan FOR UPDATE, `terpakai`, `assetBlockMessage`, `atributAssets`, `normalizeItems`, INSERT header `proses/pengajuan` + item dengan `nilai_lama`) → `upgrade.diajukan`.

### 21.2 Buat (Jalur B) — `storeAdmin`
Lock → admin → karyawan aktif → jadwal (≥ hari ini) → `insertTransaction` dengan `disetujui/menunggu_pengambilan`, jadwal, approval admin → `upgrade.disetujui`.

### 21.3 Review — `review`
Admin → header `proses` → tolak (alasan wajib → ditolak/selesai) atau setujui (asset `terpakai` di karyawan, jadwal wajib → disetujui/menunggu_pengambilan) → `upgrade.ditolak` / `upgrade.disetujui`.

### 21.4 Ubah jadwal — `updateJadwal`
Fase `menunggu_pengambilan` → kolom pengambilan; `menunggu_pengembalian` → kolom pengembalian → `upgrade.dijadwalkan`.

### 21.5 *(Dihapus v1.4)* Checklist pengambilan — `checklistPengambilan`
Method model, method controller (`checklistPengambilanUpgrade`), rute, dan notifikasi `upgrade.checklist_pengambilan` sudah dihapus. Helper bersama `Asset_m::normalizeKondisiChecklist` tetap ada karena dipakai modul lain (mis. PML01 dan Kerusakan), tetapi tidak lagi dipanggil oleh Upgrade Karyawan.

### 21.6 Serahkan — `pelepasan`
Karyawan aktif → `lockHeader` memfilter `id_karyawan` (milik orang lain → "Upgrade asset tidak ditemukan.") → fase harus `menunggu_pengambilan` (**tidak ada lagi syarat checklist**) → `lockHeldAsset` (`terpakai`/`dalam_perbaikan`, dipegang karyawan) → asset `dalam_perbaikan` + log (karyawan → gudang) → fase `dalam_service`, `tgl/user_pelepasan` → `upgrade.dilepas`.

### 21.7 Hasil — `hasil`
Admin → fase `dalam_service` → validasi status/keterangan/biaya/jadwal → UPDATE item (`nilai_baru` atau `alasan_tidak_upgrade`) → header `menunggu_pengembalian` + hasil + jadwal pengembalian, `tgl_diterima = NULL` → `upgrade.pengembalian`.

### 21.8 Terima (langsung selesai) — `terima`
Karyawan aktif → `lockHeader` memfilter `id_karyawan` → fase harus `menunggu_pengembalian` → `lockHeldAsset` (harus `dalam_perbaikan` di karyawan) → bila `status_hasil = selesai`: `applyAtribut` (update `komponen[].value` di `asset_data_item.json_value` untuk setiap `id_komponen`) → asset `terpakai` + log (gudang → karyawan) → header `fase_upgrade = selesai`, `status_transaksi = selesai`, `tgl_diterima = now` → `upgrade.diterima` ("… Upgrade selesai."). Tanpa body, tanpa checklist, satu `DB::transaction`.

### 21.9 *(Dihapus v1.4)* Review akhir — `selesai`
Method model `selesai`, method controller `selesaiUpgradeKaryawan`, rute `POST /asset/upgrade-karyawan/{id}/selesai`, aksi `terima` / `kirim_ulang`, dan notifikasi `upgrade.selesai` / `upgrade.kirim_ulang` sudah dihapus. Logika yang dulu ada di sini (`applyAtribut`, asset kembali `terpakai`, log gudang → karyawan, fase selesai) kini dijalankan oleh `terima` (21.8).

### 21.10 Query baca
- **Opsi karyawan**: asset `terpakai` miliknya, tidak diblokir flow lain, punya atribut (`atribut[]`), + daftar urgensi.
- **Opsi admin**: karyawan aktif yang memegang asset yang dapat diupgrade beserta `assets[]`.
- **List**: tab `pengajuan` (approval proses), `proses` (disetujui & fase ≠ selesai), `selesai` (`status_transaksi = selesai`, termasuk ditolak).
- **Detail**: header, jadwal, hasil, biaya, karyawan, asset, `items` (nilai lama/baru), `komponen`, `aksi`. Field `checklist_pengambilan`, `checklist_pengembalian`, `checklist_pengembalian_rusak` **tidak ada lagi**. `aksi` = `review`, `ubah_jadwal`, `pelepasan` (fase `menunggu_pengambilan`), `hasil` (fase `dalam_service`), `terima` (fase `menunggu_pengembalian`).
- **Pending**: karyawan (`pelepasan` untuk fase `menunggu_pengambilan`, `terima` untuk `menunggu_pengembalian`; tanpa syarat checklist); admin (approval `proses` ATAU fase `dalam_service`).

---

## 22. Dampak terhadap asset_data

| Field | Sebelum | Sesudah | Trigger |
|---|---|---|---|
| status_asset | terpakai | dalam_perbaikan | Serahkan aset (pelepasan) |
| status_asset | dalam_perbaikan | terpakai | Terima aset |
| pemegang_asset / id_pemegang_asset | karyawan | **tidak berubah** | Seluruh proses |
| `asset_data_item.json_value.komponen[].value` | nilai lama | nilai baru | Terima aset (hasil `selesai`) |
| tgl_update / user_update | – | now / pelaku | Setiap update |

---

## 23. API Documentation

| ID | Method | Endpoint | Function | Auth |
|---|---|---|---|---|
| API-001 | GET | `/asset/upgrade-karyawan/options` | Karyawan & asset yang dapat diupgrade | Admin |
| API-002 | GET | `/asset/upgrade-karyawan` | List | Admin |
| API-003 | POST | `/asset/upgrade-karyawan` | Buat upgrade (Jalur B) | Admin |
| API-004 | GET | `/asset/upgrade-karyawan/{id}` | Detail | Admin |
| API-005 | POST | `/asset/upgrade-karyawan/{id}/review` | Setujui / tolak | Admin |
| API-006 | PUT | `/asset/upgrade-karyawan/{id}/jadwal` | Ubah jadwal | Admin |
| API-007 | – | *(dihapus v1.4)* `/asset/upgrade-karyawan/{id}/checklist-pengambilan` | Checklist pengambilan | – |
| API-008 | POST | `/asset/upgrade-karyawan/{id}/hasil` | Input hasil + jadwal pengembalian | Admin |
| API-009 | – | *(dihapus v1.4)* `/asset/upgrade-karyawan/{id}/selesai` | Selesaikan / kirim ulang | – |
| API-010 | GET | `/asset/karyawan/upgrade/options` | Asset & atribut + urgensi | JWT |
| API-011 | GET | `/asset/karyawan/upgrade` | List milik saya | JWT |
| API-012 | POST | `/asset/karyawan/upgrade` | Ajukan (Jalur A) | JWT |
| API-013 | GET | `/asset/karyawan/upgrade/{id}` | Detail milik saya | JWT, pemilik |
| API-014 | POST | `/asset/karyawan/upgrade/{id}/pelepasan` | Serahkan aset (tanpa checklist) | JWT, pemilik |
| API-015 | POST | `/asset/karyawan/upgrade/{id}/terima` | Terima aset, **langsung menyelesaikan** (tanpa body) | JWT, pemilik |

Penomoran API tidak digeser: API-007 dan API-009 tetap tercantum sebagai baris "dihapus v1.4" agar rujukan lama (walkthrough, changelog, dokumen lain) tetap terbaca. Endpoint yang aktif **13** (admin 7: API-001 sampai 006 dan 008; karyawan 6: API-010 sampai 015), dihitung dari `API/routes/modules/asset/RouteAsset.php` (grup `upgrade-karyawan` dan grup karyawan `upgrade`). Mengirim request ke rute yang sudah dihapus tidak cocok dengan rute mana pun sehingga dijawab 404 oleh router.

Terkait: `GET /asset/karyawan/onboarding` (Asset Saya, key `upgrade`) dan `GET /asset/summary` (`upgrade_perlu_proses`, `upgrade_pending`).

---

## 24. Detail Setiap API

Header umum: `Authorization: Bearer <token>`, `Content-Type: application/json`, `Accept`, `locale`, `versi-app`. Format `{ code, message, data }`.

### API-010 — Opsi (Karyawan)
```json
{
  "code": "SUCCESS",
  "message": "Opsi upgrade asset berhasil ditemukan.",
  "data": {
    "assets": [
      {
        "id_asset_data": "CMP00120260100012",
        "kode_asset": "LPT-0012",
        "nama_asset": "Laptop 14 inch",
        "kategori_asset": "Laptop",
        "merek": "MerekX",
        "id_pemegang_asset": "KRY0010",
        "atribut": [
          {"id_atribut": "CMP001ITEM0101|ram", "id_asset_data_item": "CMP001ITEM0101", "id_komponen": "ram",
           "nama_item": "Laptop 14 inch", "jenis": "aset_utama", "label": "RAM", "satuan": "GB", "nilai": "8"}
        ]
      }
    ],
    "urgensi": [
      {"value": "normal", "label": "Normal"},
      {"value": "secepatnya", "label": "Secepatnya"},
      {"value": "mendesak", "label": "Mendesak"}
    ]
  }
}
```
API-001 (admin) mengembalikan `{ "karyawan": [ {id_karyawan, nik, nama_karyawan, assets: [...]} ] }`.

### API-012 — Ajukan (Jalur A)
```http
POST /asset/karyawan/upgrade
```
```json
{
  "id_asset_data": "CMP00120260100012",
  "alasan": "Aplikasi desain sering lambat",
  "urgensi": "secepatnya",
  "items": [
    {"id_atribut": "CMP001ITEM0101|ram", "detail_upgrade": "8 GB menjadi 16 GB"}
  ]
}
```
| Field | Rule | Message |
|---|---|---|
| id_asset_data | required\|string\|max:50 | – |
| alasan | required\|string\|max:1000 | Alasan upgrade wajib diisi. |
| urgensi | required\|in:normal,secepatnya,mendesak | Urgensi wajib dipilih. |
| items | required\|array\|min:1 | Minimal satu komponen wajib dipilih. |
| items.*.id_atribut | required\|string\|max:120\|distinct | – |
| items.*.detail_upgrade | required\|string\|max:255 | Detail upgrade setiap komponen wajib diisi. |

Pesan umum: "Data pengajuan upgrade belum lengkap."
```json
{ "code": "CREATED", "message": "Pengajuan upgrade asset berhasil dikirim.",
  "data": { "id": "CMP001202609006", "no_trx": "KRY03-2-202609006", "status_approval": "proses",
            "fase_upgrade": "pengajuan", "status_label": "Menunggu Persetujuan", "tab": "pengajuan" } }
```
422: "Karyawan tidak ditemukan atau tidak aktif.", "Asset bukan milik karyawan atau tidak ditemukan.", "Hanya asset berstatus terpakai yang dapat diupgrade.", "Asset sedang diproses pada upgrade lain." / "…laporan kerusakan." / "…retur." / "…offboarding." / "…pemeliharaan.", "Komponen upgrade tidak ditemukan pada asset.", "Komponen yang sama tidak boleh dipilih lebih dari sekali." 500: "Terjadi kesalahan saat menyimpan pengajuan upgrade."

### API-003 — Buat (Jalur B)
```http
POST /asset/upgrade-karyawan
```
```json
{
  "id_karyawan": "KRY0010",
  "id_asset_data": "CMP00120260100012",
  "catatan": "Program peremajaan laptop tim desain",
  "items": [{"id_atribut": "CMP001ITEM0101|ram", "detail_upgrade": "8 GB menjadi 16 GB"}],
  "tgl_jadwal": "2026-09-30",
  "jam_jadwal": "10:00",
  "keterangan_jadwal": "Bawa ke ruang IT"
}
```
Rules: `id_karyawan` required max 150, `id_asset_data` required, `catatan` nullable max 1000, items seperti API-012, jadwal `tgl_jadwal` (Y-m-d) / `jam_jadwal` (H:i) / `keterangan_jadwal` (≤255). Pesan umum: "Data upgrade asset belum lengkap." Response `CREATED` "Upgrade asset berhasil dibuat." (`disetujui` / `menunggu_pengambilan`). 500: "Terjadi kesalahan saat membuat upgrade asset."

### API-002 / API-011 — List
`tab`: `pengajuan` | `proses` | `selesai`; `limit` 1–50. Record: `id, no_trx, sumber_pengajuan, tgl_pengajuan, urgensi, status_approval, fase_upgrade, status_label, tab, tgl/jam_pengambilan, tgl/jam_pengembalian, jumlah_item, asset{nama_asset, kategori_asset}, karyawan{id_karyawan, nik, nama_karyawan}` + `filters`, `pagination`.

### API-004 / API-013 — Detail
```json
{
  "id": "CMP001202609006",
  "no_trx": "KRY03-2-202609006",
  "sumber_pengajuan": "karyawan",
  "tgl_pengajuan": "2026-09-28",
  "alasan": "Aplikasi desain sering lambat",
  "urgensi": "secepatnya",
  "catatan": "",
  "ket_tolak": "",
  "status_approval": "disetujui",
  "fase_upgrade": "selesai",
  "status_label": "Selesai",
  "tab": "selesai",
  "tgl_pengambilan": "2026-09-30",
  "jam_pengambilan": "10:00:00",
  "tgl_pengembalian": "2026-10-03",
  "jam_pengembalian": "15:00:00",
  "keterangan_jadwal": "",
  "tgl_pelepasan": "2026-09-30 10:12:00",
  "tgl_diterima": "2026-10-03 15:20:00",
  "status_hasil": "selesai",
  "keterangan_hasil": "RAM ditambah 1 keping 8 GB",
  "biaya_upgrade": 650000,
  "karyawan": {"id_karyawan": "KRY0010", "nik": "EMP-010", "nama_karyawan": "Dimas Contoh", "cabang": "Kantor Pusat", "divisi": "Kreatif"},
  "asset": {"id_asset_data": "CMP00120260100012", "kode_asset": "LPT-0012", "nama_asset": "Laptop 14 inch", "kategori_asset": "Laptop", "merek": "MerekX", "status_asset": "terpakai"},
  "items": [
    {"id_item": "CMP001202609014", "nama_item": "Laptop 14 inch", "label": "RAM", "satuan": "GB",
     "detail_upgrade": "8 GB menjadi 16 GB", "nilai_lama": "8", "nilai_baru": "16", "alasan_tidak_upgrade": ""}
  ],
  "komponen": [{"id_item": "CMP001ITEM0101", "nama_item": "Laptop 14 inch", "jenis": "aset_utama", "detail": "RAM: 8 GB"}],
  "aksi": {"review": false, "ubah_jadwal": false, "pelepasan": false, "hasil": false, "terima": false}
}
```
404: "Upgrade asset tidak ditemukan."

Contoh di atas adalah transaksi yang sudah `selesai` (semua `aksi` bernilai `false`). Sejak v1.4 detail **tidak lagi memuat** `checklist_pengambilan`, `checklist_pengembalian`, `checklist_pengembalian_rusak`, dan `aksi` hanya berisi `review`, `ubah_jadwal`, `pelepasan` (fase `menunggu_pengambilan`), `hasil` (fase `dalam_service`), `terima` (fase `menunggu_pengembalian`).

### API-005 — Review
```json
{ "keputusan": "disetujui", "ket_tolak": "", "tgl_jadwal": "2026-09-30", "jam_jadwal": "10:00", "keterangan_jadwal": "" }
```
Rules: `keputusan` required in:disetujui,ditolak; `ket_tolak` required_if ditolak (≤255, "Alasan penolakan wajib diisi."). "Review upgrade asset berhasil disimpan." 422: "Pengajuan upgrade sudah direview.", "Status asset sudah berubah atau tidak lagi dipegang karyawan.", "Tanggal dan jam serah terima wajib diisi."

### API-006 — Ubah Jadwal
```json
{ "tgl_jadwal": "2026-10-01", "jam_jadwal": "09:00", "keterangan_jadwal": "" }
```
"Jadwal upgrade berhasil diperbarui." 422: "Jadwal hanya dapat diubah sebelum serah terima dilakukan."

### API-007 — Checklist Pengambilan (Admin) — DIHAPUS v1.4
`POST /asset/upgrade-karyawan/{id}/checklist-pengambilan` tidak ada lagi di `RouteAsset.php`; request ke rute ini dijawab 404. Nomor API-007 dipertahankan agar penomoran lain tidak bergeser.

### API-014 — Serahkan Aset (Karyawan)
Tanpa body dan **tanpa syarat checklist** (v1.4). Karyawan harus pemilik transaksi dan fase `menunggu_pengambilan`. "Aset berhasil diserahkan untuk upgrade." 422: "Aset belum siap diserahkan untuk upgrade.", "Karyawan tidak ditemukan atau tidak aktif.", "Status asset sudah berubah atau tidak lagi dipegang karyawan.", "Upgrade asset tidak ditemukan." (transaksi milik karyawan lain / PIC).

### API-015 — Terima Aset (Karyawan) — langsung menyelesaikan
```http
POST /asset/karyawan/upgrade/{id}/terima
```
**Tanpa body dan tanpa checklist** (v1.4); bila body dikirim, isinya diabaikan. Satu request menyelesaikan transaksi: atribut diperbarui (bila `status_hasil = selesai`), asset `terpakai`, log gudang → karyawan, `fase_upgrade = selesai`, `status_transaksi = selesai`, `tgl_diterima` terisi.
```json
{ "code": "UPDATED", "message": "Aset hasil upgrade berhasil diterima.",
  "data": { "id": "CMP001202610001", "no_trx": "KRY03-2-202610001", "status_approval": "disetujui",
            "fase_upgrade": "selesai", "status_label": "Selesai", "tab": "selesai" } }
```
422: "Aset belum dijadwalkan untuk dikembalikan." (fase bukan `menunggu_pengembalian`, termasuk pemanggilan kedua), "Status asset sudah berubah atau tidak lagi dipegang karyawan.", "Karyawan tidak ditemukan atau tidak aktif.", "Upgrade asset tidak ditemukan." (milik karyawan lain / PIC). Notifikasi `upgrade.diterima` ke admin.

### API-008 — Hasil Upgrade (Admin)
```json
{
  "status_hasil": "selesai",
  "keterangan_hasil": "RAM ditambah 1 keping 8 GB",
  "biaya_upgrade": 650000,
  "items": [{"id_item": "CMP001202609014", "nilai_baru": "16"}],
  "tgl_jadwal": "2026-10-03",
  "jam_jadwal": "15:00",
  "keterangan_jadwal": ""
}
```
| Field | Rule |
|---|---|
| status_hasil | required\|in:selesai,tidak_dapat_diupgrade |
| keterangan_hasil | required\|string\|max:1000 |
| biaya_upgrade | nullable\|integer\|min:0 |
| items | nullable\|array; `items.*.id_item` required distinct; `items.*.nilai_baru` nullable ≤255 |
| tgl/jam/keterangan_jadwal | jadwal pengembalian |

"Hasil upgrade berhasil disimpan." 422: "Aset belum berada dalam proses upgrade.", "Nilai baru setiap atribut wajib diisi.", "Tanggal dan jam serah terima wajib diisi."

### API-009 — Review Akhir (Admin) — DIHAPUS v1.4
`POST /asset/upgrade-karyawan/{id}/selesai` (aksi `terima` / `kirim_ulang`) tidak ada lagi; request ke rute ini dijawab 404. Penyelesaian transaksi kini dilakukan oleh API-015. Nomor API-009 dipertahankan agar penomoran lain tidak bergeser.

---

## 25. Mobile Architecture

```text
Page (admin: bool) → AssetUpgradeBloc → AssetUpgradeApi(admin) → ApiClient → Backend
```
Satu set layar dipakai admin dan karyawan; `AssetUpgradeApi(admin: true)` memakai base `asset/upgrade-karyawan`, selain itu `asset/karyawan/upgrade`.

| Layer | File | Function |
|---|---|---|
| Api | `Repository/Service/assetUpgradeApi.dart` | getList, getOptions, getDetail, store, review, updateJadwal, hasil, pelepasan(id), terima(id) tanpa body (memakai `AssetKerusakanJadwal` untuk jadwal; file yang sama juga memuat `storeRuangan` untuk modul Upgrade Ruangan) |
| Model | `Repository/Model/Asset/mAssetUpgrade.dart` | AssetUpgradeFase, AssetUpgradeAtribut, AssetUpgradeAsset, AssetUpgradeKaryawanOption, AssetUpgradeItem, AssetUpgradeAksi, AssetUpgradeRecord, AssetUpgradePage, AssetUpgradeOptions |
| Bloc | `Bloc/Asset/AssetUpgradeBloc.dart` | LoadAssetUpgradeList, LoadMoreAssetUpgradeList, LoadAssetUpgradeOptions, LoadAssetUpgradeDetail, SubmitAssetUpgradeAction |
| UI | `UI/Modul/Asset/Upgrade/*` | HalamanAssetUpgrade, HalamanTambahAssetUpgrade, HalamanDetailAssetUpgrade, AssetUpgradeWidgets |

| Route | Page |
|---|---|
| `/asset_upgrade` · `/asset_admin_upgrade` | List (karyawan / admin) |
| `/asset_tambah_upgrade` · `/asset_admin_tambah_upgrade` | Ajukan / Buat Upgrade Asset |
| `/asset_detail_upgrade` · `/asset_admin_detail_upgrade` | Detail |

---

## 26. Mobile Screen

### 26.1 Upgrade Asset — list
Tab Pengajuan / Proses / Selesai; kartu dengan status (Menunggu Persetujuan, Menunggu Pengambilan, Dalam Service, Menunggu Pengembalian, Selesai, Ditolak), label "Diinisiasi Admin/HR" untuk Jalur B. Label "Menunggu Review" masih ada di kode hanya untuk menampilkan data lama. Kosong: "Belum ada data upgrade asset."

### 26.2 Ajukan Upgrade Asset (karyawan) / Buat Upgrade Asset (admin)
| Field UI | Karyawan | Admin | Catatan |
|---|:---:|:---:|---|
| Pilih Karyawan ("Ketuk untuk memilih karyawan") | – | ✅ | Dari opsi admin |
| Pilih Asset ("Ketuk untuk memilih asset") | ✅ | ✅ | "Tidak ada aset yang dapat diupgrade." |
| Komponen yang Diupgrade | ✅ | ✅ | "Pilih spesifikasi yang ingin diupgrade, lalu tulis target spesifikasinya."; tiap atribut "Saat ini: {nilai}" + **Detail Upgrade** ("Contoh: 8 GB menjadi 16 GB") |
| Alasan | ✅ | – | "Alasan pengajuan wajib diisi." |
| Urgensi (Normal / Secepatnya / Mendesak) | ✅ | – | |
| Catatan Tambahan | – | ✅ | |
| Jadwal Pengambilan | – | ✅ | Sheet tanggal & jam |
| Tombol | **Ajukan Upgrade** | **Buat Upgrade** | |

### 26.3 Detail Upgrade Asset
Stepper **Pengajuan → Pengambilan → Service → Pengembalian → Selesai** (tidak berubah; data lama fase `menunggu_review` ditampilkan pada langkah Pengembalian); info asset, karyawan, Jadwal Pengambilan / Jadwal Pengembalian ("{tanggal}, jam {jam}"), **Komponen yang Diupgrade** (Sebelum / Sesudah), **Hasil Upgrade** (Upgrade Selesai / Tidak Dapat Diupgrade, Keterangan Hasil, Biaya Upgrade (Rp)). **Tidak ada lagi form atau tampilan checklist** (pengambilan maupun penerimaan).

| Kondisi (`aksi`) | Role | Tampilan / Tombol |
|---|---|---|
| review | Admin | **Setujui & Jadwalkan** / **Tolak** (+Alasan Penolakan) |
| pelepasan | Karyawan | Info "Admin/HR sudah memeriksa kondisi aset. Konfirmasi setelah aset diserahkan." (teks lama masih tampil, lihat INC-10) → **Serahkan Aset** (dialog "Pastikan aset sudah diserahkan kepada Admin/HR.") |
| fase `dalam_service` | Admin & karyawan | Info "Aset sedang dalam proses upgrade oleh teknisi." |
| hasil | Admin | Form hasil (status, Nilai Baru per atribut bila selesai, Keterangan Hasil, Biaya) → **Simpan Hasil & Jadwalkan** |
| terima | Karyawan | **Terima Aset** (dialog "Pastikan aset hasil upgrade sudah Anda terima dan periksa."); tanpa form, langsung menyelesaikan |
| ubah_jadwal | Admin | **Ubah Jadwal** |

Tombol **Simpan Checklist Pengambilan**, **Selesaikan Upgrade**, dan **Kirim Ulang ke Teknisi** sudah dihapus dari layar. Tombol Serahkan / Terima kini mengikuti `aksi.pelepasan` / `aksi.terima`.

### 26.4 Entry point
Asset Saya: menu **Upgrade** + banner **Serahkan Aset untuk Upgrade** / **Terima Aset Hasil Upgrade** (banner pelepasan muncul sejak fase `menunggu_pengambilan`). Asset Karyawan: aksi **Upgrade** + panel **Upgrade Perlu Diproses** ("Lihat semua ({jumlah})"; isinya approval `proses` atau fase `dalam_service`).

---

## 27. Mobile → API

| UI Field | JSON Field | Type | Required |
|---|---|---|:---:|
| Pilih Karyawan (admin) | `id_karyawan` | string | Jalur B |
| Pilih Asset | `id_asset_data` | string | Ya |
| Atribut dicentang | `items[].id_atribut` | `id_asset_data_item|id_komponen` | Min 1 |
| Detail Upgrade | `items[].detail_upgrade` | string ≤255 | Ya |
| Alasan | `alasan` | string ≤1000 | Jalur A |
| Urgensi | `urgensi` | normal/secepatnya/mendesak | Jalur A |
| Catatan Tambahan | `catatan` | string ≤1000 | Tidak |
| Jadwal | `tgl_jadwal`, `jam_jadwal`, `keterangan_jadwal` | Y-m-d, H:i, string | Jalur B, setujui, hasil, ubah jadwal |
| Setujui / Tolak | `keputusan`, `ket_tolak` | string | Review |
| Hasil | `status_hasil`, `keterangan_hasil`, `biaya_upgrade`, `items[].id_item`, `items[].nilai_baru` | – | Input hasil |
| Serahkan Aset | – (tanpa body) | – | `pelepasan(id)` |
| Terima Aset | – (tanpa body) | – | `terima(id)` |

Sejak v1.4 tidak ada lagi payload checklist (`items[].kondisi`, `ket_rusak`) maupun `aksi` review akhir (`terima`/`kirim_ulang`).

---

## 28. API → Mobile

| API JSON | Mobile Model | UI |
|---|---|---|
| `records[]` | AssetUpgradePage / AssetUpgradeRecord | Kartu list |
| `fase_upgrade`, `status_label` | AssetUpgradeRecord / AssetUpgradeFase | Badge, stepper |
| `sumber_pengajuan` | AssetUpgradeRecord | "Diinisiasi Admin/HR" |
| `items[]` (nilai lama/baru) | AssetUpgradeItem | Sebelum / Sesudah |
| `komponen` | AssetUpgradeRecord | Masih di-parse, tidak lagi dipakai UI Upgrade Karyawan (sisa) |
| `status_hasil`, `keterangan_hasil`, `biaya_upgrade` | AssetUpgradeRecord | Hasil Upgrade |
| `aksi` (`review`, `ubah_jadwal`, `pelepasan`, `hasil`, `terima`) | AssetUpgradeAksi | Tombol |
| `assets[].atribut` / `karyawan[].assets` | AssetUpgradeAsset / Atribut / KaryawanOption | Form |

Field lama `checklist_pengambilan`, `checklist_pengembalian`, `checklist_pengembalian_rusak`, dan `aksi.checklist_pengambilan` / `aksi.selesai` diabaikan oleh model (ada test yang memastikan payload lama tetap bisa di-parse).

---

## 29. Mapping Mobile → API → Database

| UI | JSON | Backend | Table | Column |
|---|---|---|---|---|
| Asset | `id_asset_data` | `insertTransaction` | item | id_asset_data + snapshot |
| Atribut | `items[].id_atribut` | `normalizeItems` | item | kelengkapan_item (JSON), nilai_lama |
| Detail Upgrade | `items[].detail_upgrade` | trim ≤255 | item / header | detail_upgrade / ringkasan |
| Alasan / Urgensi | `alasan`, `urgensi` | `storeKaryawan` | header | alasan, urgensi |
| Catatan | `catatan` | `storeAdmin` | header | catatan |
| Jadwal | `tgl_jadwal`, `jam_jadwal`, `keterangan_jadwal` | `parseJadwalValue` | header | tgl/jam_pengambilan atau tgl/jam_pengembalian, keterangan_pengembalian |
| Serahkan Aset | – | `pelepasan()` | asset_data / asset_trx_log / header | status_asset / log / fase_upgrade, tgl_pelepasan, user_pelepasan |
| Hasil | `status_hasil`, `keterangan_hasil`, `biaya_upgrade`, `items[].nilai_baru` | `hasil()` | header / item | status_hasil, keterangan_hasil, biaya_upgrade / nilai_baru, alasan_tidak_upgrade |
| Terima Aset | – | `terima()` + `applyAtribut` | asset_data_item / asset_data / asset_trx_log / header | json_value / status_asset / log / fase_upgrade = selesai, status_transaksi = selesai, tgl_diterima |

Kolom `checklist_pengambilan` dan `checklist_pengembalian` **tidak lagi dipetakan** dari UI mana pun (tidak ditulis/dibaca sejak v1.4).

---

## 30. Mapping Database → API → Mobile

| Table | Column | API JSON | UI |
|---|---|---|---|
| header | status_approval + fase_upgrade | `status_label`, `tab`, `aksi` | Badge, tab, tombol |
| header | jadwal (nol → '') | `tgl/jam_pengambilan`, `tgl/jam_pengembalian` | Jadwal |
| header | checklist_pengambilan, checklist_pengembalian | – (tidak dikirim sejak v1.4) | – |
| item | kelengkapan_item | `items[].nama_item`, `label`, `satuan` | Nama atribut |
| item | nilai_lama / nilai_baru | idem | Sebelum / Sesudah |
| asset_data_item | json_value | `komponen[]` | Tidak dipakai UI Upgrade Karyawan (sisa) |

---

## 31. Validation

### 31.1 Mobile
| Lokasi | Aturan | Pesan |
|---|---|---|
| Form | Asset wajib; min 1 komponen | "Pilih minimal satu komponen." |
| Form | Detail upgrade per komponen | "Detail upgrade wajib diisi." |
| Form karyawan | Alasan wajib | "Alasan pengajuan wajib diisi." |
| Form admin | Karyawan wajib; jadwal | – |
| Hasil | Nilai baru bila selesai; keterangan wajib | "Nilai baru wajib diisi." / "Keterangan hasil wajib diisi." |

### 31.2 Backend — §9 & §24.
### 31.3 Database — enum `sumber_pengajuan`, `fase_upgrade`, `status_hasil`, `status_approval`, `status`; `jenis_log` + `upgrade_karyawan`.

---

## 32. Error Handling

| Error | HTTP | Backend | Mobile |
|---|---:|---|---|
| Field tidak valid | 422 | VALIDATION_ERROR | Pesan field |
| Aturan bisnis | 422 | BUSINESS_RULE_VIOLATION | Snackbar |
| Tidak ditemukan | 404 | NOT_FOUND ("Upgrade asset tidak ditemukan.") | "Detail upgrade asset tidak dapat dimuat." |
| Token / bukan admin | 401 / 403 | UNAUTHORIZED / FORBIDDEN | Pesan |
| Exception | 500 | SYSTEM_ERROR ("Terjadi kesalahan saat memproses transaksi asset.") + log | Snackbar |

---

## 33. Audit Trail

| Data | Disimpan? | Lokasi |
|---|:---:|---|
| Pengaju / pembuat | ✅ | `user_input`, `sumber_pengajuan` |
| Keputusan | ✅ | `user_approval`, `tgl_approval`, `ket_tolak` |
| Kondisi sebelum & sesudah | ❌ | **Tidak dicatat sejak v1.4** (checklist dihapus). Kolom `checklist_pengambilan` / `checklist_pengembalian` hanya menyimpan sisa data lama; `asset_trx_log` selalu menulis `kondisi_akhir = kondisi_awal` |
| Nilai atribut lama/baru | ✅ | item `nilai_lama`, `nilai_baru` |
| Hasil & biaya | ✅ | `status_hasil`, `keterangan_hasil`, `biaya_upgrade` |
| Waktu serah dan terima | ✅ | `tgl_pelepasan` / `user_pelepasan`; `tgl_diterima` (juga waktu selesai) |
| Perpindahan asset | ✅ | `asset_trx_log upgrade_karyawan` (karyawan → gudang saat serah, gudang → karyawan saat terima) |
| Perubahan `asset_data_item` | ⚠️ | Hanya `tgl_update/user_update` pada baris item (pelaku = karyawan yang menekan Terima) |

---

## 34. Notification

| Event | Trigger | Penerima | Judul | Pesan | Rute |
|---|---|---|---|---|---|
| `upgrade.diajukan` | Ajukan (Jalur A) | Admin | Pengajuan Upgrade Asset | {nama} mengajukan upgrade asset {no_trx}. | `asset_admin_detail_upgrade` |
| `upgrade.ditolak` | Tolak | Karyawan | Upgrade Asset Ditolak | Pengajuan upgrade {no_trx} ditolak admin. | `asset_detail_upgrade` |
| `upgrade.disetujui` | Setujui / Buat (Jalur B) | Karyawan | Jadwal Pengambilan Upgrade | Upgrade {no_trx} disetujui. Cek jadwal pengambilan asset. | `asset_detail_upgrade` |
| `upgrade.dijadwalkan` | Ubah jadwal | Karyawan | Jadwal Upgrade Diperbarui | Jadwal upgrade {no_trx} telah diperbarui. | `asset_detail_upgrade` |
| `upgrade.dilepas` | Serahkan aset | Admin | Asset Diserahkan untuk Upgrade | {nama} telah menyerahkan asset {no_trx} untuk diupgrade. | `asset_admin_detail_upgrade` |
| `upgrade.pengembalian` | Input hasil | Karyawan | Jadwal Pengembalian Asset | Hasil upgrade {no_trx} sudah tersedia. Cek jadwal pengembalian. | `asset_detail_upgrade` |
| `upgrade.diterima` | Terima aset (langsung selesai) | Admin | Asset Upgrade Diterima | {nama} telah menerima asset upgrade {no_trx}. Upgrade selesai. | `asset_admin_detail_upgrade` |

Event yang **dihapus v1.4**: `upgrade.checklist_pengambilan`, `upgrade.selesai`, `upgrade.kirim_ulang`. Karena karyawan menyelesaikan transaksi sendiri lewat Terima, karyawan tidak lagi menerima notifikasi "Upgrade Asset Selesai". Daftar di atas diambil dari `AssetNotification.php` (kunci `upgrade.*`; kunci `upgrade_ruangan.*` milik modul lain).

---

## 35. Security

| Aspek | Implementasi |
|---|---|
| Autentikasi | Bearer JWT |
| Isolasi company | `id_company` dari token |
| Otorisasi admin | `accessError` + `actorAdmin` |
| Kepemilikan | Karyawan: filter `id_karyawan` (list, detail, serahkan, terima); asset harus dipegang karyawan transaksi. Karyawan lain / PIC tidak dapat menyerahkan atau menerima upgrade milik orang lain (`lockHeader` memfilter `id_karyawan`) |
| Karyawan aktif | Dicek saat ajukan, buat (Jalur B), serahkan, terima; tidak di opsi/list/detail |
| Integritas | Lock company, FOR UPDATE header/item/asset/asset_data_item |

---

## 36. Manual Book User/Admin

> Label dari `assets/id/asset.json` (`menu_asset.upgrade`). Screenshot menyusul.

### A. Karyawan — Mengajukan Upgrade
1. **Asset Saya** → **Upgrade** → **Ajukan Upgrade**.
2. **Pilih Asset**, centang spesifikasi pada **Komponen yang Diupgrade**, isi **Detail Upgrade** (contoh: "8 GB menjadi 16 GB").
3. Isi **Alasan**, pilih **Urgensi** → **Ajukan Upgrade**.

### B. Admin — Membuat Upgrade Langsung
**Upgrade** → **Buat Upgrade** → **Pilih Karyawan** → **Pilih Asset** → komponen + detail → **Catatan Tambahan** → **Jadwal Pengambilan** → **Buat Upgrade**.

### C. Admin — Menyetujui
Detail → **Setujui & Jadwalkan** (isi jadwal pengambilan) atau **Tolak** (isi alasan).

### D. Pengambilan (Karyawan)
Pada jadwal pengambilan, karyawan membuka banner **Serahkan Aset untuk Upgrade** (atau detail upgrade) → **Serahkan Aset** → konfirmasi. **Tidak ada checklist** yang harus diisi admin maupun karyawan (v1.4); tombol langsung aktif begitu status **Menunggu Pengambilan**.

### E. Hasil Upgrade (Admin)
Detail → pilih **Upgrade Selesai** (isi **Nilai Baru** tiap atribut) atau **Tidak Dapat Diupgrade** → **Keterangan Hasil** → **Biaya Upgrade (Rp)** → jadwal pengembalian → **Simpan Hasil & Jadwalkan**.

### F. Pengembalian (Karyawan)
Terima aset dari teknisi/admin, periksa sendiri kondisinya, lalu buka banner **Terima Aset Hasil Upgrade** (atau detail) → **Terima Aset** → konfirmasi. **Tidak ada checklist.** Pada saat itu juga upgrade **selesai**: spesifikasi asset diperbarui (bila hasil berhasil) dan aset kembali aktif (`terpakai`).

### G. Bila ada kerusakan saat serah terima
Kondisi komponen tidak lagi dicatat di Upgrade. Bila karyawan menemukan kerusakan setelah menerima aset, laporkan lewat fitur **Kerusakan** (KRY06). Tidak ada tombol Kirim Ulang ke Teknisi.

---

## 37. Kondisi Button

| Button | Role | Condition (`aksi`) | Action |
|---|---|---|---|
| Ajukan Upgrade | Karyawan | Form valid | POST karyawan/upgrade |
| Buat Upgrade | Admin | Form valid | POST upgrade-karyawan |
| Setujui & Jadwalkan / Tolak | Admin | `review` | POST review |
| Ubah Jadwal | Admin | `ubah_jadwal` | PUT jadwal |
| Serahkan Aset | Karyawan | `pelepasan` (fase `menunggu_pengambilan`) | POST pelepasan |
| Simpan Hasil & Jadwalkan | Admin | `hasil` (fase `dalam_service`) | POST hasil |
| Terima Aset | Karyawan | `terima` (fase `menunggu_pengembalian`) | POST terima (langsung selesai) |
| Simpan Checklist Pengambilan | – | **Dihapus v1.4** | – |
| Selesaikan Upgrade / Kirim Ulang ke Teknisi | – | **Dihapus v1.4** | – |
| Batalkan | – | ⚠️ Belum ditemukan | – |

---

## 38. Testing Scenario

Test otomatis PHPUnit (`API/tests/Asset/AssetUpgradeKaryawanTest.php`): **5 test, 54 assertion**, lulus pada 2026-10-30 (`vendor/bin/phpunit tests/Asset/AssetUpgradeKaryawanTest.php`, PHP 8.2.31, PHPUnit 10.5.45). Isi: `test_options_list_spec_attributes_of_held_assets`, `test_employee_request_is_stored_and_can_be_rejected`, `test_full_upgrade_updates_attribute_and_returns_asset`, `test_failed_upgrade_keeps_attribute`, `test_employee_route_submits_request_and_notifies_admin`.

Test otomatis Flutter (`MOBILE/test/asset/asset_upgrade_test.dart`): **5 test**, lulus pada 2026-10-30 (`flutter test --no-pub`, Flutter 3.35.6): teks lokalisasi id/en lengkap; detail mem-parse item dan aksi serta mengabaikan field checklist lama; beranda dan summary menampilkan upgrade yang perlu tindakan; form ajukan mewajibkan detail komponen dan alasan; admin pada fase service wajib mengisi nilai baru per komponen.

| TC | Scenario | Expected |
|---|---|---|
| TC-01 | Karyawan ajukan upgrade valid | 201; Menunggu Persetujuan; notif admin |
| TC-02 | Tanpa alasan / urgensi salah | 422 |
| TC-03 | Atribut bukan milik asset / duplikat | 422 |
| TC-04 | Asset sedang upgrade/kerusakan/retur/offboarding/pemeliharaan | 422 |
| TC-05 | Admin tolak tanpa alasan | 422 |
| TC-06 | Admin tolak | Ditolak (tab selesai); asset tidak berubah |
| TC-07 | Admin setujui tanpa jadwal | 422 |
| TC-08 | Jalur B dengan jadwal kemarin | 422 |
| TC-09 | Serahkan di luar fase `menunggu_pengambilan` (mis. sudah `dalam_service`) | 422 "Aset belum siap diserahkan untuk upgrade." |
| TC-10 | Serahkan / terima oleh karyawan lain atau PIC (bukan pemilik transaksi) | 422 "Upgrade asset tidak ditemukan." (`lockHeader` memfilter `id_karyawan`) |
| TC-11 | Serahkan aset **tanpa checklist** (karyawan, fase `menunggu_pengambilan`, tanpa body) | asset `dalam_perbaikan` (pemegang tetap karyawan); log karyawan → gudang; Dalam Service; notif admin |
| TC-12 | Hasil selesai tanpa nilai baru | 422 |
| TC-13 | Hasil tidak dapat diupgrade | `alasan_tidak_upgrade` terisi; Menunggu Pengembalian |
| TC-14 | Terima **tanpa checklist** (hasil `selesai`) | Langsung Selesai: `asset_data_item` bernilai baru; asset `terpakai`; log gudang → karyawan; `status_transaksi = selesai`; `tgl_diterima` terisi; notif admin "… Upgrade selesai." |
| TC-15 | Terima (hasil `tidak_dapat_diupgrade`) | Selesai; atribut tidak berubah; asset `terpakai`; log tetap dibuat |
| TC-16 | Terima dua kali | Kedua kali ditolak 422 "Aset belum dijadwalkan untuk dikembalikan." (fase sudah `selesai`); tidak ada perubahan data |
| TC-17 | Terima sebelum hasil dicatat (fase `dalam_service`) | 422 "Aset belum dijadwalkan untuk dikembalikan." |
| TC-18 | Ubah jadwal saat dalam_service | 422 |
| TC-19 | Asset dalam upgrade diajukan Retur / di-Offboarding | Keduanya ditolak "…sedang diproses pada upgrade." |
| TC-20 | Asset dalam upgrade dilaporkan rusak | Ditolak Kerusakan |
| TC-21 | Tanpa token / bukan admin | 401 / 403 |
| TC-22 | Rute lama `POST /asset/upgrade-karyawan/{id}/checklist-pengambilan` dan `POST /asset/upgrade-karyawan/{id}/selesai` | 404 (rute tidak ada lagi) |
| TC-23 | Detail upgrade (admin/karyawan) | Tidak memuat `checklist_pengambilan`, `checklist_pengembalian`, `checklist_pengembalian_rusak`; `aksi` hanya `review`, `ubah_jadwal`, `pelepasan`, `hasil`, `terima` |
| TC-24 | Data lama fase `menunggu_review` | Semua `aksi = false`; asset tetap `dalam_perbaikan`; setelah skrip SQL dijalankan fase menjadi `selesai` |

Cakupan: TC-01, TC-03, TC-04, TC-06, TC-11, TC-12, TC-13, TC-14, TC-15, TC-20 dan perilaku `aksi` (TC-23 sebagian) diverifikasi oleh test PHPUnit di atas (alur penuh: aksi pelepasan sejak `menunggu_pengambilan`, pending admin hanya pada `dalam_service`, terima langsung `selesai`, log 2 baris). **TC-16, TC-17, TC-22, TC-24 diturunkan dari kode dan routing, belum ada test otomatisnya**; TC-24 bergantung pada skrip SQL yang belum dijalankan.

---

## 39. Temuan Ketidaksesuaian

| ID | Component | Finding | Impact | Recommendation |
|---|---|---|---|---|
| INC-01 | DBML ↔ Migration/Code | DBML header memakai index milik tabel peminjaman (`jenis_status`, `status`, `id_asset_trx_peminjaman_karyawan`); item DBML punya `review_admin`, `review_karyawan`, `konfirmasi_admin` yang tidak dipakai, tanpa `nilai_lama/nilai_baru`; kolom workflow tidak ada; format `no_trx` "KRY-03-2-" vs kode "KRY03-2-"; kolom `checklist_*` belum ditandai tidak terpakai | Skema tidak akurat | Perbarui DBML |
| INC-02 | Backend | List admin upgrade memakai `validateRusakListRequest` → pesan error "Filter laporan kerusakan tidak valid." / "Parameter daftar laporan kerusakan tidak valid." | Pesan salah konteks | Buat validasi list upgrade sendiri |
| INC-03 | Backend | Log serah terima mencatat `posisi_akhir = gudang` / `pemegang_asset = gudang`, padahal `asset_data.pemegang_asset` tetap karyawan | Log dan master berbeda | Samakan (catat posisi fisik secara eksplisit) |
| INC-04 | Backend | Satu kolom `keterangan_pengembalian` dipakai untuk catatan jadwal pengambilan dan pengembalian (API: `keterangan_jadwal`) | Catatan tertimpa | Pisahkan kolom |
| INC-05 | Backend | *(Tertutup v1.4)* Admin dapat Selesaikan walau checklist penerimaan berisi item rusak | Tidak relevan lagi: checklist dan review akhir dihapus | – |
| INC-06 | Backend | *(Tertutup v1.4)* Setelah kirim ulang, checklist, hasil, dan biaya sebelumnya tertimpa | Tidak relevan lagi: kirim ulang dihapus | – |
| INC-07 | Mobile | Model jadwal upgrade memakai `AssetKerusakanJadwal` (modul Kerusakan) | Coupling antar modul | Model bersama / sendiri |
| INC-08 | Backend | Validasi request sebelum cek admin | 422 bukan 403 | Pindahkan cek |
| INC-09 | Backend | Tab `selesai` mencampur Ditolak dan Selesai | Riwayat tercampur | Pisahkan filter |
| INC-10 | Mobile | Teks `pelepasan_info` ("Admin/HR sudah memeriksa kondisi aset. Konfirmasi setelah aset diserahkan.") masih tampil pada fase `menunggu_pengambilan` padahal admin tidak lagi memeriksa apa pun | Pesan menyesatkan karyawan | Ubah teks di `asset.json` (assets/id dan assets/en) |
| INC-11 | Mobile / Backend | Sisa kode: kunci lokalisasi `menu_asset.upgrade.*` terkait checklist/selesaikan/kirim ulang (mis. `checklist_pengambilan`, `button_simpan_checklist`, `button_selesaikan`, `button_kirim_ulang`, `review_normal_info`) masih ada di `asset.json` (assets/id dan assets/en); API masih mengirim `komponen[]` yang kini tidak dipakai layar Upgrade Karyawan; konstanta/label `menunggu_review` dipertahankan untuk data lama | Beban pemeliharaan kecil | Bersihkan setelah skrip SQL data lama dijalankan |
| INC-12 | Database | Kolom `checklist_pengambilan` dan `checklist_pengembalian` tidak ditulis/dibaca lagi tetapi tetap ada; enum `fase_upgrade` tetap memuat `menunggu_review` | Skema memuat kolom/nilai mati | Hapus kolom dan nilai enum setelah data lama selesai (butuh persetujuan migrasi) |

---

## 40. Edge Case

| Edge Case | Current Behavior | Impact | Recommendation |
|---|---|---|---|
| Asset `dalam_perbaikan` (di teknisi) diretur | Ditolak "…pada upgrade." | Aman | – |
| Asset dalam upgrade di-offboarding | Ditolak (v1.1) | Aman | – |
| Karyawan nonaktif sebelum serahkan/terima | `activeKaryawan` gagal | Transaksi macet | Aksi admin pengganti |
| Upgrade disetujui tapi tidak pernah diambil | Menunggu Pengambilan selamanya | Asset terkunci dari flow lain | Batal / kedaluwarsa |
| Karyawan menekan Terima sebelum fisik asset kembali | Langsung selesai; atribut berubah dan tidak ada verifikasi admin | Data spesifikasi bisa mendahului kenyataan | Pertimbangkan konfirmasi admin atau jeda |
| Terima ditekan dua kali (mis. koneksi lambat) | Request kedua ditolak 422 "Aset belum dijadwalkan untuk dikembalikan."; data tidak berubah | Aman | – |
| Aset rusak saat diterima | Tidak ada tempat mencatatnya di Upgrade; transaksi tetap selesai | Kondisi tak tercatat | Laporkan lewat Kerusakan (KRY06) |
| Transaksi lama di `menunggu_review` | Tidak ada aksi; asset tetap `dalam_perbaikan` | Asset terkunci | Jalankan skrip SQL (RISK-008) |
| Atribut dihapus dari `asset_data_item` sebelum Terima | `applyAtribut` melewati baris yang tidak ada | Nilai baru tidak tersimpan tanpa error | Validasi saat Terima |
| Nilai baru dikirim untuk `id_item` yang tidak ada | Diabaikan | – | Validasi id item |

---

## 41. Potential Bug / Technical Risk

### RISK-001 — Retur dan Offboarding mengecek upgrade aktif (SELESAI)
**Lokasi:** `hasActiveUpgradeAsset` dipanggil `AssetRusakKaryawan_m`, `AssetPemeliharaanKaryawan_m`, dan — sejak v1.1 — `AssetOffboardingKaryawan_m::assertNoCrossFlow` ("Satu atau beberapa asset sedang diproses pada upgrade."). `AssetReturKaryawan_m::store` kini juga memanggilnya ("Satu atau beberapa asset sedang diproses pada upgrade.").
**Masalah:** Sebelumnya asset `dalam_perbaikan` (di teknisi, pemegang tetap karyawan) masih dapat diretur; kini ditolak saat ajukan retur.
**Recommendation:** Tidak ada; risiko ditutup (terverifikasi 12 Okt 2026).

### RISK-002 — Status asset tidak mencerminkan posisi fisik (SEDANG)
Pemegang tetap karyawan selama service. (Kasus status `dalam_perbaikan` setelah kirim ulang sudah tidak ada sejak v1.4.)

### RISK-003 — Tidak ada pembatalan / kedaluwarsa (SEDANG)
Setelah disetujui, upgrade tidak dapat dibatalkan; asset terus terkunci dari flow lain.

### RISK-004 — Pembaruan atribut tanpa jejak terpisah (SEDANG)
`applyAtribut` (kini dipanggil saat karyawan menekan Terima) menimpa `json_value`; riwayat hanya ada di item transaksi (nilai lama/baru), bukan di log asset.

### RISK-005 — Tanggal nol `0000-00-00` (SEDANG)
⚠️ Asumsi: bermasalah pada MySQL strict mode.

### RISK-006 — Notifikasi di luar transaksi (RENDAH)
Dikirim setelah commit; kegagalan tidak membatalkan transaksi.

### RISK-007 — Kapasitas penomoran (RENDAH)
3 digit/bulan untuk header & item (satu item per atribut).

### RISK-008 — Data lama di `menunggu_review` terkunci sampai SQL dijalankan (TINGGI, ketergantungan rilis)
Setelah v1.4 tidak ada aksi untuk transaksi yang sudah berada di fase `menunggu_review` (review akhir dihapus), sehingga asetnya tetap `dalam_perbaikan` dan terblokir dari flow lain. Penyelesaiannya satu kali lewat `Claude outputs/sql-upgrade-hapus-checklist-review-lama.sql` (menerapkan atribut bila hasil `selesai`, asset `terpakai`, log gudang → karyawan/ruangan, fase `selesai`; mencakup Upgrade Karyawan dan Upgrade Ruangan). **Skrip BELUM dijalankan dan sintaksnya belum diuji di DB**; jalankan dulu di salinan/staging, periksa guard 0.c, lalu COMMIT. Rilis v1.4 sebaiknya menunggu keputusan menjalankan skrip ini.

### RISK-009 — Kondisi saat serah terima tidak lagi tercatat (SEDANG)
Tanpa checklist, kerusakan yang muncul sebelum atau sesudah upgrade tidak punya bukti di sistem; log aset selalu `kondisi_akhir = kondisi_awal`. Sengketa tanggung jawab antara karyawan, admin, dan teknisi tidak dapat dirunut dari data.

### RISK-010 — Penyelesaian sepihak oleh karyawan (SEDANG)
Satu klik **Terima Aset** langsung mengubah spesifikasi asset dan status; tidak ada verifikasi admin. Nilai baru berasal dari input admin pada langkah Hasil, jadi risiko terutama pada waktu penerimaan (karyawan menekan Terima sebelum asset benar-benar kembali).

---

## 42. Dependency

```text
Asset terpakai milik karyawan + atribut spesifikasi di asset_data_item
      ↓
Upgrade (Jalur A / B → pengambilan → service → pengembalian → selesai) ──► Notifikasi
      ↓
asset_data (dalam_perbaikan ↔ terpakai), asset_data_item (nilai baru), asset_trx_log upgrade_karyawan

Aturan silang:
  - Upgrade mengecek: upgrade lain, Kerusakan, Retur, Offboarding, Pemeliharaan PML01
  - Dicek oleh: Kerusakan, Offboarding, Pemeliharaan PML01, Retur (hasActiveUpgradeAsset)
  - Summary admin: upgrade_perlu_proses / upgrade_pending; Asset Saya: upgrade[]
  - Helper bersama Asset_m::normalizeKondisiChecklist dan widget AssetKondisiChecklist tetap ada (dipakai modul lain), tidak dipakai Upgrade Karyawan

Ketergantungan rilis v1.4:
  - Skrip sql-upgrade-hapus-checklist-review-lama.sql (BELUM dijalankan) untuk transaksi lama di menunggu_review (Upgrade Karyawan dan Ruangan)
  - Aplikasi mobile lama yang masih memanggil .../checklist-pengambilan atau .../selesai menerima 404
```

---

## 43. Source Code Reference

| File | Layer | Function |
|---|---|---|
| `API/routes/modules/asset/RouteAsset.php` (grup `upgrade-karyawan` baris 101–108; grup karyawan `upgrade` baris 241–247) | Route | 13 endpoint (7 admin + 6 karyawan) |
| `API/app/Http/Controllers/Asset/Admin/AssetAdminUpgrade_c.php` | Controller | Admin upgrade, `UPGRADE_*_RULES` |
| `API/app/Http/Controllers/Asset/Karyawan/AssetKaryawanUpgrade_c.php` | Controller | Karyawan upgrade |
| `API/app/Models/Asset/AssetUpgradeKaryawan_m.php` | Model | Seluruh logika |
| `API/app/Models/Asset/Asset_m.php` | Model | Helper bersama |
| `API/app/Models/Asset/AssetRusakKaryawan_m.php`, `AssetOffboardingKaryawan_m.php`, `AssetPemeliharaanKaryawan_m.php` | Model | Pemakai `hasActiveUpgradeAsset` |
| `API/app/Models/Asset/AssetKaryawan_m.php`, `AssetAdmin_m.php` | Model | Pending karyawan & summary |
| `API/app/Support/Asset/AssetNotification.php` | Support | `upgrade.*` |
| `API/database/migrations/2026_09_26_000001_add_workflow_fields_to_asset_upgrade_karyawan.php` | Migration | Kolom workflow, index, log |
| `Claude outputs/sql-upgrade-hapus-checklist-review-lama.sql` | SQL | Penyelesaian satu kali transaksi lama `menunggu_review` (**belum dijalankan**) |
| `API/tests/Asset/AssetUpgradeKaryawanTest.php` | Test | 5 test, 54 assertion |
| `MOBILE/test/asset/asset_upgrade_test.dart` | Test | 5 test |
| `MOBILE/lib/Repository/Service/assetUpgradeApi.dart` | Api | – |
| `MOBILE/lib/Repository/Model/Asset/mAssetUpgrade.dart` | Model | – |
| `MOBILE/lib/Bloc/Asset/AssetUpgradeBloc.dart` | Bloc | – |
| `MOBILE/lib/UI/Modul/Asset/Upgrade/*` | UI | – |

---

## 44. Developer Notes

- **Satu upgrade = satu asset**; item = atribut (`id_atribut = id_asset_data_item|id_komponen`).
- **Atribut** diambil dari `json_value.komponen[]` dengan `flag_spesifikasi` 1/2, kecuali field bertipe `file`.
- **`applyAtribut`** mengelompokkan item per `id_asset_data_item` dan menulis `value` untuk setiap `id_komponen` yang punya nilai baru. Sejak v1.4 dipanggil dari `terima()` (bukan lagi dari `selesai()`), dengan pelaku karyawan.
- **Satu set layar mobile** untuk admin & karyawan (`admin: bool`); tombol ditentukan objek `aksi` (`review`, `ubah_jadwal`, `pelepasan`, `hasil`, `terima`).
- **`AssetUpgradeApi.terima(id)`** tidak lagi mengirim body; `pelepasan(id)` tetap tanpa body.
- **Widget `AssetKondisiChecklist`** dipertahankan karena dipakai modul lain; Upgrade Karyawan tidak memakainya.
- **Jangan mengubah kolom/enum** `checklist_*` dan `menunggu_review` tanpa persetujuan migrasi; data lama masih memakainya.
- **Nomor transaksi** memakai sufiks tipe `-2` (tipe Upgrade Komponen); tipe KRY03 lainnya ⚠️ Belum ditemukan di kode.

---

## 45. End-to-End Traceability

```text
Karyawan: Laptop → atribut RAM (Saat ini: 8) → "8 GB menjadi 16 GB" → Ajukan Upgrade
  → POST /asset/karyawan/upgrade → storeKaryawan() → header KRY03-2-… (proses/pengajuan) + item nilai_lama "8"
Admin: Setujui & Jadwalkan 23 Okt 10:00 → review() → disetujui/menunggu_pengambilan
Karyawan: Serahkan Aset (tanpa checklist) → pelepasan() → asset dalam_perbaikan + log → dalam_service
Admin: Upgrade Selesai, Nilai Baru "16", biaya 650.000, jadwal 26 Okt → hasil() → item nilai_baru "16" → menunggu_pengembalian
Karyawan: Terima Aset (tanpa checklist) → terima() → asset_data_item RAM value "16", asset terpakai + log → selesai
  → Mobile: Sebelum 8 / Sesudah 16, badge "Selesai"
```

### 45.4 Walkthrough Lengkap dengan Contoh Data

Bagian ini mengikuti **satu upgrade dari pengajuan sampai atribut diperbarui**, langkah demi langkah: apa yang dilakukan pengguna di mobile, apa yang dikirim, apa yang diproses API, dan apa yang tersimpan di database. Data bersifat **sintetis**, tetapi format ID, nama kolom, urutan proses, label status, dan pesan diambil dari kode (`AssetUpgradeKaryawan_m`, `AssetKaryawanUpgrade_c`, `AssetAdminUpgrade_c`, `AssetNotification`). Sejak v1.4 walkthrough ini **6 langkah (langkah 0 sampai 5)** karena checklist dan review akhir sudah tidak ada.

**Skenario (Jalur A):** Dimas mengajukan upgrade RAM laptopnya dari 8 GB ke 16 GB. Admin menyetujui dan menjadwalkan pengambilan, Dimas menyerahkan laptop (tanpa checklist), admin mencatat hasil upgrade beserta jadwal pengembalian, lalu Dimas menekan **Terima Aset** (tanpa checklist) dan upgrade langsung selesai: nilai RAM baru ditulis ke data komponen.

| Langkah | Pelaku | Aksi | Endpoint |
|:---:|---|---|---|
| 0 | Karyawan | Memuat asset dan atribut yang bisa diupgrade | API-010 `GET /asset/karyawan/upgrade/options` |
| 1 | Karyawan | **Ajukan Upgrade** | API-012 `POST /asset/karyawan/upgrade` |
| 2 | Admin | **Setujui** + jadwal pengambilan | API-005 `POST /asset/upgrade-karyawan/{id}/review` |
| 3 | Karyawan | **Serahkan Aset** (tanpa checklist) | API-014 `POST /asset/karyawan/upgrade/{id}/pelepasan` |
| 4 | Admin | **Hasil Upgrade** + jadwal pengembalian | API-008 `POST /asset/upgrade-karyawan/{id}/hasil` |
| 5 | Karyawan | **Terima Aset** (tanpa checklist, langsung selesai) | API-015 `POST /asset/karyawan/upgrade/{id}/terima` |

#### Data awal

Company `CMP001`. Pengajuan 22 Oktober 2026, periode ID `202610`. Belum ada upgrade, log, atau notifikasi pada periode ini.

**`data_karyawan`**

| id_karyawan | nik | nama_lengkap | Peran |
|---|---|---|---|
| KRY0010 | EMP-010 | Dimas Contoh | Pemohon, pemegang asset |
| KRY0099 | ADM-001 | Siti Admin | Admin Asset (aktor) |
| KRY0100 | ADM-002 | Rudi Admin | Admin Asset lain (ikut menerima notifikasi) |

**`asset_data`**: A12 `CMP00120260100012` (LPT-0012), Laptop 14 inch, kategori Laptop (`CMP001202601002`), `status_asset = terpakai`, `pemegang_asset = karyawan`, `id_pemegang_asset = KRY0010`, `kondisi_awal = normal`.

**`asset_data_item`** milik A12 (`json_value`, hanya yang `flag_spesifikasi` 1 atau 2 yang bisa dipilih)

| id_asset_data_item | nama_item | flag_spesifikasi | komponen (`json_value.komponen[]`) |
|---|---|:---:|---|
| CMP001ITEM0101 | Laptop 14 inch | 1 (unit utama) | `{"id":"ram","label":"RAM","satuan":"GB","value":"8"}` |
| CMP001ITEM0102 | Charger | 2 (kelengkapan) | `{"id":"daya","label":"Daya","satuan":"W","value":"65"}` |

Setiap field `komponen` (kecuali bertipe `file`) menjadi satu **atribut** dengan `id_atribut = id_asset_data_item|id_komponen`, misalnya `CMP001ITEM0101|ram`.

Penomoran: header dan item `id_company + YYYYMM + 3 digit`, `no_trx = KRY03-2-YYYYMMNNN`; log dan notifikasi 5 digit. Selama proses, **`pemegang_asset` tetap karyawan**; hanya `status_asset` yang berubah (`dalam_perbaikan`).

---

#### Langkah 0 — Karyawan membuka form

**Mobile:** Asset Saya → **Upgrade** → **+** → form **Ajukan Upgrade Asset** → `AssetUpgradeApi.getOptions()`.

**Dikirim**
```http
GET /asset/karyawan/upgrade/options
Authorization: Bearer <access_token_dimas>
```

**Diproses API** (`AssetUpgradeKaryawan_m::getOptions`): asset milik karyawan yang `terpakai`, tidak diblokir flow lain, dan **punya atribut spesifikasi** (`atributAssets`); ditambah daftar urgensi.

**Database:** hanya `SELECT`.

**Diterima mobile** (ringkas)
```json
{ "code": "SUCCESS", "message": "Opsi upgrade asset berhasil ditemukan.",
  "data": {
    "assets": [ { "id_asset_data": "CMP00120260100012", "kode_asset": "LPT-0012", "nama_asset": "Laptop 14 inch",
      "kategori_asset": "Laptop", "id_pemegang_asset": "KRY0010",
      "atribut": [
        { "id_atribut": "CMP001ITEM0101|ram", "id_asset_data_item": "CMP001ITEM0101", "id_komponen": "ram",
          "nama_item": "Laptop 14 inch", "jenis": "aset_utama", "label": "RAM", "satuan": "GB", "nilai": "8" },
        { "id_atribut": "CMP001ITEM0102|daya", "id_asset_data_item": "CMP001ITEM0102", "id_komponen": "daya",
          "nama_item": "Charger", "jenis": "kelengkapan", "label": "Daya", "satuan": "W", "nilai": "65" } ] } ],
    "urgensi": [ { "value": "normal", "label": "Normal" }, { "value": "secepatnya", "label": "Secepatnya" },
                 { "value": "mendesak", "label": "Mendesak" } ] } }
```
Unit utama selalu ditampilkan lebih dulu daripada kelengkapan.

#### Langkah 1 — Dimas mengajukan upgrade

**Mobile:** Dimas memilih laptop, mencentang atribut **RAM**, mengisi detail "8 GB menjadi 16 GB", alasan, dan urgensi **Secepatnya**, lalu **Ajukan Upgrade** → `AssetUpgradeApi.storeKaryawan`.

**Dikirim**
```http
POST /asset/karyawan/upgrade
```
```json
{
  "id_asset_data": "CMP00120260100012",
  "alasan": "Aplikasi desain sering lambat",
  "urgensi": "secepatnya",
  "items": [ { "id_atribut": "CMP001ITEM0101|ram", "detail_upgrade": "8 GB menjadi 16 GB" } ]
}
```

**Diproses API** (`AssetKaryawanUpgrade_c::storeUpgradeSaya` → `storeKaryawan` → `insertTransaction`, satu `DB::transaction`)
1. Validasi: `alasan` wajib (maks 1000); `urgensi` ∈ normal/secepatnya/mendesak; `items` minimal 1; `id_atribut` unik; `detail_upgrade` wajib (maks 255).
2. `lockCompany`; `activeKaryawan("KRY0010")`.
3. Asset diambil `FOR UPDATE`: dipegang Dimas dan **`terpakai`**; `assetBlockMessage` memastikan tidak ada upgrade, kerusakan, retur, offboarding, atau pemeliharaan aktif.
4. `atributAssets` + `normalizeItems`: setiap atribut harus milik asset itu dan tidak boleh duplikat.
5. Generate ID: header urutan 1 → `CMP001202610001`, `no_trx = KRY03-2-202610001`.
6. INSERT header (`sumber_pengajuan = karyawan`, `status_approval = proses`, `fase_upgrade = pengajuan`, jadwal nol) dan INSERT item per atribut (snapshot JSON dan `nilai_lama`).
7. **`asset_data` tidak berubah.** Notifikasi `upgrade.diajukan` ke seluruh admin.

**Hasil database**

`asset_trx_upgrade_karyawan`

| Kolom | Nilai |
|---|---|
| id_asset_trx_upgrade_karyawan | CMP001202610001 |
| no_trx | KRY03-2-202610001 |
| sumber_pengajuan | **karyawan** |
| id_asset_kategori_m / nama_kategori | CMP001202601002 / Laptop |
| detail_upgrade | RAM: 8 GB menjadi 16 GB |
| alasan / urgensi / catatan | Aplikasi desain sering lambat / secepatnya / `''` |
| id_karyawan / nik_karyawan / nama_karyawan | KRY0010 / EMP-010 / Dimas Contoh |
| tgl_pengajuan | 2026-10-22 |
| tgl_pengambilan / jam_pengambilan / tgl_pengembalian / jam_pengembalian | 0000-00-00 / 00:00:00 / 0000-00-00 / 00:00:00 |
| status_approval / fase_upgrade / status_transaksi | **proses** / **pengajuan** / proses |
| user_approval / tgl_approval | `''` / NULL |
| checklist_pengambilan / checklist_pengembalian | NULL (kolom tidak dipakai sejak v1.4) |
| tgl_input / user_input | 2026-10-22 09:00:00 / EMP-010 - Dimas Contoh |

`asset_trx_upgrade_karyawan_item`

| Kolom | Nilai |
|---|---|
| id_asset_trx_upgrade_item | CMP001202610001 |
| id_asset_trx_upgrade / id_asset_data | CMP001202610001 / A12 |
| kelengkapan_item | `{"id_atribut":"CMP001ITEM0101\|ram","id_asset_data_item":"CMP001ITEM0101","id_komponen":"ram","nama_item":"Laptop 14 inch","jenis":"aset_utama","label":"RAM","satuan":"GB","nilai":"8"}` |
| detail_upgrade | 8 GB menjadi 16 GB |
| nilai_lama / nilai_baru / alasan_tidak_upgrade | 8 / `''` / `''` |

`notif_data` (2 baris, satu per admin)

| id_notif | id_karyawan | id_ref | rute | ket_notif_id |
|---|---|---|---|---|
| CMP00120261000001 | KRY0099 | CMP001202610001 | asset_admin_detail_upgrade | Dimas Contoh mengajukan upgrade asset KRY03-2-202610001. |
| CMP00120261000002 | KRY0100 | CMP001202610001 | asset_admin_detail_upgrade | Dimas Contoh mengajukan upgrade asset KRY03-2-202610001. |

**Diterima mobile** (HTTP 201)
```json
{ "code": "CREATED", "message": "Pengajuan upgrade asset berhasil dikirim.",
  "data": { "id": "CMP001202610001", "no_trx": "KRY03-2-202610001", "status_approval": "proses",
            "fase_upgrade": "pengajuan", "status_label": "Menunggu Persetujuan", "tab": "pengajuan" } }
```

#### Langkah 2 — Admin menyetujui dan menjadwalkan pengambilan

**Mobile:** notifikasi membuka detail admin (`aksi.review = true`). Admin memilih **Setujui**, mengisi tanggal 23 Okt 2026 pukul 10:00 dan catatan → `AssetUpgradeAdminApi.review`.

**Dikirim**
```http
POST /asset/upgrade-karyawan/CMP001202610001/review
```
```json
{ "keputusan": "disetujui", "ket_tolak": "", "tgl_jadwal": "2026-10-23", "jam_jadwal": "10:00",
  "keterangan_jadwal": "Bawa ke ruang IT" }
```

**Diproses API** (`reviewUpgradeKaryawan` → `review`)
1. Validasi: `keputusan` ∈ disetujui/ditolak; `ket_tolak` wajib bila ditolak. Cek admin, `lockCompany`.
2. Header harus `status_approval = proses` ("Pengajuan upgrade sudah direview.").
3. **Setuju:** asset masih `terpakai` dan dipegang Dimas (`lockHeldAsset`, "Status asset sudah berubah atau tidak lagi dipegang karyawan."); jadwal wajib `Y-m-d` + `H:i`, tanggal tidak boleh sebelum `tgl_pengajuan`.
4. UPDATE header: `status_approval = disetujui`, `fase_upgrade = menunggu_pengambilan`, jadwal pengambilan, `keterangan_pengembalian` (kolom catatan untuk **kedua** jadwal), `user_approval`, `tgl_approval`, `tgl_update`, `user_update`.
5. Notifikasi `upgrade.disetujui` ke karyawan.

**Hasil database**

| Tabel | Perubahan |
|---|---|
| header | status_approval `proses → disetujui`; fase_upgrade `pengajuan → menunggu_pengambilan`; tgl_pengambilan 2026-10-23; jam_pengambilan 10:00:00; keterangan_pengembalian = Bawa ke ruang IT; user_approval = ADM-001 - Siti Admin; tgl_approval = 2026-10-22 10:00:00; tgl_update/user_update sama |
| asset_data A12 | **Tidak berubah** |
| notif_data | + `CMP00120261000003`: KRY0010, id_ref `CMP001202610001`, rute `asset_detail_upgrade`, "Upgrade KRY03-2-202610001 disetujui. Cek jadwal pengambilan asset." |

**Diterima mobile** (HTTP 200): `status_approval = disetujui`, `fase_upgrade = menunggu_pengambilan`, `status_label = "Menunggu Pengambilan"`, `tab = "proses"`, message "Review upgrade asset berhasil disimpan.". Sejak saat ini `aksi.pelepasan = true` bagi Dimas (tanpa menunggu checklist). Admin masih bisa **Ubah Jadwal** (API-006) selama fase ini; yang berubah adalah kolom jadwal pengambilan, dengan notifikasi `upgrade.dijadwalkan`.

#### Langkah 3 — Dimas menyerahkan aset (tanpa checklist)

**Mobile:** banner di Asset Saya atau detail upgrade → **Serahkan Aset** → dialog konfirmasi → `AssetUpgradeApi.pelepasan(id)`. Tidak ada form checklist.

**Dikirim**
```http
POST /asset/karyawan/upgrade/CMP001202610001/pelepasan
```
Tanpa body.

**Diproses API** (`pelepasanUpgradeSaya` → `pelepasan`)
1. Karyawan aktif; `lockHeader` memfilter `id_karyawan` sehingga hanya header milik Dimas yang ditemukan; fase harus `menunggu_pengambilan` ("Aset belum siap diserahkan untuk upgrade."). **Tidak ada syarat checklist.**
2. Asset dikunci dan harus masih dipegang Dimas (`terpakai` atau `dalam_perbaikan`).
3. UPDATE `asset_data` A12 → **`dalam_perbaikan`** (`pemegang_asset` **tidak diubah**). INSERT `asset_trx_log` (`jenis_log = upgrade_karyawan`, karyawan → gudang).
4. UPDATE header: `fase_upgrade = dalam_service`, `tgl_pelepasan`, `user_pelepasan`.
5. Notifikasi `upgrade.dilepas` ke admin.

**Hasil database**

| Tabel | Perubahan |
|---|---|
| asset_data A12 | status_asset `terpakai → dalam_perbaikan`; pemegang_asset tetap karyawan; id_pemegang_asset tetap KRY0010; tgl_update = 2026-10-23 10:15:00; user_update = EMP-010 - Dimas Contoh |
| header | fase_upgrade `menunggu_pengambilan → dalam_service`; tgl_pelepasan = 2026-10-23 10:15:00; user_pelepasan = EMP-010 - Dimas Contoh |

`asset_trx_log` (1 baris)

| Kolom | Nilai |
|---|---|
| id_asset_trx_log | CMP00120261000001 |
| id_asset_data / jenis_log | A12 / upgrade_karyawan |
| id_transaksi / nama_tabel_transaksi | CMP001202610001 / asset_trx_upgrade_karyawan |
| kondisi_awal / kondisi_akhir | normal / normal (selalu sama; kondisi saat serah terima tidak dicatat sejak v1.4) |
| posisi_awal / posisi_akhir | karyawan / gudang |
| pemegang_asset / id_pemegang / nama_pemegang | gudang / KRY0010 / Dimas Contoh |
| user_input | EMP-010 - Dimas Contoh |

`notif_data`: `CMP00120261000004` (KRY0099) dan `CMP00120261000005` (KRY0100), rute `asset_admin_detail_upgrade`, "Dimas Contoh telah menyerahkan asset KRY03-2-202610001 untuk diupgrade."

**Diterima mobile** (HTTP 200): `fase_upgrade = dalam_service`, `status_label = "Dalam Service"`, message "Aset berhasil diserahkan untuk upgrade.".

#### Langkah 4 — Admin mencatat hasil upgrade

**Mobile:** teknisi menyelesaikan pekerjaan di luar sistem. Admin membuka **Hasil Upgrade**, memilih **Selesai**, mengisi keterangan, biaya, nilai baru RAM, dan jadwal pengembalian 26 Okt 2026 pukul 15:00 → `AssetUpgradeAdminApi.hasil`.

**Dikirim**
```http
POST /asset/upgrade-karyawan/CMP001202610001/hasil
```
```json
{
  "status_hasil": "selesai",
  "keterangan_hasil": "RAM ditambah 1 keping 8 GB",
  "biaya_upgrade": 650000,
  "items": [ { "id_item": "CMP001202610001", "nilai_baru": "16" } ],
  "tgl_jadwal": "2026-10-26", "jam_jadwal": "15:00", "keterangan_jadwal": ""
}
```

**Diproses API** (`hasilUpgradeKaryawan` → `hasil`)
1. Fase harus `dalam_service` ("Aset belum berada dalam proses upgrade.").
2. `status_hasil` ∈ selesai/tidak_dapat_diupgrade; `keterangan_hasil` wajib (maks 1000); `biaya_upgrade` ≥ 0; jadwal pengembalian wajib.
3. Per item: bila `selesai`, `nilai_baru` wajib ("Nilai baru setiap atribut wajib diisi."); bila `tidak_dapat_diupgrade`, `nilai_baru` dikosongkan dan `alasan_tidak_upgrade` diisi dari keterangan (maks 255).
4. UPDATE header: `fase_upgrade = menunggu_pengembalian`, hasil, biaya, jadwal pengembalian, `keterangan_pengembalian`, serta **reset** `tgl_diterima = NULL`.
5. Notifikasi `upgrade.pengembalian` ke karyawan.

**Hasil database**

| Tabel | Perubahan |
|---|---|
| item `…001` | nilai_baru `'' → 16`; alasan_tidak_upgrade `''` |
| header | fase_upgrade `dalam_service → menunggu_pengembalian`; status_hasil = selesai; keterangan_hasil = RAM ditambah 1 keping 8 GB; biaya_upgrade = 650000; tgl_pengembalian 2026-10-26; jam_pengembalian 15:00:00; tgl_diterima NULL |
| asset_data A12 | Tidak berubah (`dalam_perbaikan`) |
| notif_data | + `CMP00120261000006`: KRY0010, rute `asset_detail_upgrade`, "Hasil upgrade KRY03-2-202610001 sudah tersedia. Cek jadwal pengembalian." |

**Diterima mobile** (HTTP 200): `fase_upgrade = menunggu_pengembalian`, `status_label = "Menunggu Pengembalian"`, message "Hasil upgrade berhasil disimpan.". `aksi.terima = true` bagi Dimas. Ubah Jadwal pada fase ini mengubah jadwal **pengembalian**.

#### Langkah 5 — Dimas menerima laptop (langsung selesai)

**Mobile:** Dimas menerima laptop dan memeriksanya sendiri, lalu **Terima Aset** → dialog konfirmasi → `AssetUpgradeApi.terima(id)`. Tidak ada form checklist dan tidak ada body.

**Dikirim**
```http
POST /asset/karyawan/upgrade/CMP001202610001/terima
```
Tanpa body.

**Diproses API** (`terimaUpgradeSaya` → `terima`, satu `DB::transaction`)
1. Karyawan aktif; `lockHeader` memfilter `id_karyawan`; fase harus `menunggu_pengembalian` ("Aset belum dijadwalkan untuk dikembalikan.").
2. Asset harus `dalam_perbaikan` dan masih dipegang Dimas ("Status asset sudah berubah atau tidak lagi dipegang karyawan.").
3. Bila `status_hasil = selesai` → **`applyAtribut`**: untuk tiap item, `asset_data_item.json_value.komponen[]` dengan `id` = `id_komponen` diperbarui `value`-nya ke `nilai_baru`, bila tidak kosong.
4. UPDATE `asset_data` A12 → **`terpakai`**; INSERT `asset_trx_log` (gudang → karyawan).
5. UPDATE header: `fase_upgrade = selesai`, `status_transaksi = selesai`, `tgl_diterima = now`.
6. Notifikasi `upgrade.diterima` ke admin ("… Upgrade selesai."). Karyawan tidak menerima notifikasi tambahan.

**Hasil database**

| Tabel | Perubahan |
|---|---|
| asset_data_item `CMP001ITEM0101` | `json_value.komponen[id=ram].value`: **8 → 16**; tgl_update/user_update = EMP-010 - Dimas Contoh |
| asset_data A12 | status_asset `dalam_perbaikan → terpakai`; tgl_update = 2026-10-26 15:20:00; user_update = EMP-010 - Dimas Contoh |
| header | fase_upgrade `menunggu_pengembalian → selesai`; status_transaksi `proses → selesai`; tgl_diterima = 2026-10-26 15:20:00; tgl_update = 2026-10-26 15:20:00; user_update = EMP-010 - Dimas Contoh |

`asset_trx_log` (1 baris baru): `CMP00120261000002`, A12, `upgrade_karyawan`, kondisi normal → normal, posisi **gudang → karyawan**, id_pemegang KRY0010, nama_pemegang Dimas Contoh, `user_input` = EMP-010 - Dimas Contoh.

`notif_data`: `CMP00120261000007` (KRY0099) dan `CMP00120261000008` (KRY0100), rute `asset_admin_detail_upgrade`, "Dimas Contoh telah menerima asset upgrade KRY03-2-202610001. Upgrade selesai."

**Diterima mobile** (HTTP 200): `fase_upgrade = selesai`, `status_label = "Selesai"`, `tab = "selesai"`, message "Aset hasil upgrade berhasil diterima.". Semua nilai `aksi` menjadi `false`.

---

#### Kondisi akhir data

`asset_trx_upgrade_karyawan`: `CMP001202610001` / `KRY03-2-202610001`, `status_approval = disetujui`, `fase_upgrade = selesai`, `status_transaksi = selesai`, `status_hasil = selesai`, `biaya_upgrade = 650000`, `tgl_pelepasan = 2026-10-23 10:15:00`, `tgl_diterima = 2026-10-26 15:20:00`, `checklist_pengambilan` dan `checklist_pengembalian` tetap NULL.

`asset_trx_upgrade_karyawan_item`: `CMP001202610001`, `nilai_lama = 8`, `nilai_baru = 16`.

`asset_data`: A12 → `terpakai` @KRY0010 (tidak pernah berpindah pemegang). `asset_data_item`: RAM = 16. `asset_trx_log`: 2 baris. `notif_data`: 8 baris.

---

#### Variasi

**Jalur B (admin menginisiasi).** `POST /asset/upgrade-karyawan` dengan `id_karyawan`, `id_asset_data`, `items[]`, `catatan`, dan jadwal pengambilan (**wajib, tanggal ≥ hari ini**). Header langsung `sumber_pengajuan = admin`, `status_approval = disetujui`, `fase_upgrade = menunggu_pengambilan`, `alasan` dan `urgensi` kosong, `user_approval` = admin pembuat; notifikasi `upgrade.disetujui` ke karyawan. Langkah 3 sampai 5 sama.

**Admin menolak.** Pada langkah 2, `keputusan = ditolak` dengan `ket_tolak` (wajib): `status_approval = ditolak`, `status_transaksi = selesai`, `fase_upgrade = selesai`, asset tidak berubah, notifikasi `upgrade.ditolak`.

**Hasil `tidak_dapat_diupgrade`.** Pada langkah 4, `status_hasil = tidak_dapat_diupgrade` (nilai baru tidak diisi):
- item: `nilai_baru` dikosongkan (`''`) dan `alasan_tidak_upgrade` diisi dari `keterangan_hasil` (maks 255 karakter); header `status_hasil = tidak_dapat_diupgrade`, `biaya_upgrade` tetap boleh diisi, jadwal pengembalian tetap wajib, fase `menunggu_pengembalian`.
- Pada langkah 5, `applyAtribut` **dilewati** sehingga `asset_data_item` (RAM tetap 8) tidak berubah; namun asset tetap kembali `terpakai`, log gudang → karyawan tetap dibuat, header menjadi `selesai` dengan `tgl_diterima` terisi, dan notifikasi `upgrade.diterima` tetap berbunyi "… Upgrade selesai.".
- Tidak ada lagi siklus kirim ulang: bila upgrade harus diulang, pengajuan baru harus dibuat.

---

#### Kasus gagal yang umum

| Situasi | Langkah | HTTP | Pesan | Perubahan database |
|---|---|:---:|---|---|
| Asset bukan milik karyawan | 1 | 422 | Asset bukan milik karyawan atau tidak ditemukan. | Tidak ada |
| Asset bukan `terpakai` | 1 | 422 | Hanya asset berstatus terpakai yang dapat diupgrade. | Tidak ada |
| Asset di flow lain | 1 | 422 | Asset sedang diproses pada upgrade lain. / …laporan kerusakan. / …retur. / …offboarding. / …pemeliharaan. | Tidak ada |
| Atribut bukan milik asset / duplikat | 1 | 422 | Komponen upgrade tidak ditemukan pada asset. / Komponen yang sama tidak boleh dipilih lebih dari sekali. | Tidak ada |
| Alasan, urgensi, atau detail kosong | 1 | 422 | Alasan upgrade wajib diisi. / Urgensi wajib dipilih. / Detail upgrade setiap komponen wajib diisi. | Tidak ada |
| Review dua kali | 2 | 422 | Pengajuan upgrade sudah direview. | Tidak ada |
| Asset sudah berubah status atau pemegang | 2, 3, 5 | 422 | Status asset sudah berubah atau tidak lagi dipegang karyawan. | Tidak ada |
| Jadwal kosong / format salah / sebelum tanggal pengajuan | 2, 4 | 422 | Tanggal dan jam serah terima wajib diisi. / Format jadwal serah terima tidak valid. / Tanggal serah terima tidak boleh sebelum tanggal pengajuan. | Tidak ada |
| Ubah jadwal di luar fase yang diizinkan | – | 422 | Jadwal hanya dapat diubah sebelum serah terima dilakukan. | Tidak ada |
| Serahkan di luar fase `menunggu_pengambilan` | 3 | 422 | Aset belum siap diserahkan untuk upgrade. | Tidak ada |
| Karyawan lain / PIC menyerahkan atau menerima milik orang lain | 3, 5 | 422 | Upgrade asset tidak ditemukan. | Tidak ada |
| Hasil di luar fase `dalam_service` | 4 | 422 | Aset belum berada dalam proses upgrade. | Tidak ada |
| Nilai baru kosong saat hasil `selesai` | 4 | 422 | Nilai baru setiap atribut wajib diisi. | Tidak ada |
| Keterangan hasil kosong / biaya negatif | 4 | 422 | Keterangan hasil upgrade wajib diisi. / Biaya upgrade tidak boleh negatif. | Tidak ada |
| Terima sebelum dijadwalkan kembali, atau terima kedua kali | 5 | 422 | Aset belum dijadwalkan untuk dikembalikan. | Tidak ada |
| Memanggil rute lama `…/checklist-pengambilan` atau `…/selesai` | – | 404 | Rute tidak ada (dihapus v1.4) | Tidak ada |
| Pengguna bukan admin asset | 2, 4 | 403 / 422 | Pengguna tidak memiliki akses admin asset. / Pengguna tidak terdaftar sebagai admin asset. | Tidak ada |
| Karyawan tidak aktif | 1, 3, 5 | 422 | Karyawan tidak ditemukan atau tidak aktif. | Tidak ada |
| Token kedaluwarsa | semua | 401 | Token telah expired. Mobile me-refresh token sekali lalu mengulang request. | Tidak ada |

Semua pelanggaran aturan dilempar sebagai `InvalidArgumentException` di dalam `DB::transaction`, sehingga seluruh perubahan request itu dibatalkan. Pesan tersedia dalam bahasa Indonesia dan Inggris mengikuti header `locale`.

#### Ringkasan siapa mengerjakan apa

| Tahap | Mobile | API | Database |
|---|---|---|---|
| Ajukan / Buat | Pilih asset, atribut, detail, alasan, urgensi (Jalur A); atau karyawan + jadwal (Jalur B) | Validasi atribut, cek flow lain, snapshot nilai lama | header, item (`nilai_lama`), `notif_data` |
| Review dan jadwal | Setujui/tolak, tanggal dan jam | Validasi jadwal, set fase | header, `notif_data` |
| Serahkan | Tombol Serahkan Aset (tanpa checklist) | Asset → `dalam_perbaikan`, log, fase `dalam_service` | `asset_data`, `asset_trx_log`, header, `notif_data` |
| Hasil | Status hasil, biaya, nilai baru, jadwal kembali | Simpan nilai baru, reset `tgl_diterima` | item, header, `notif_data` |
| Terima (langsung selesai) | Tombol Terima Aset (tanpa checklist) | Tulis nilai baru ke komponen (bila berhasil), asset → `terpakai`, log, fase `selesai` | `asset_data_item`, `asset_data`, `asset_trx_log`, header, `notif_data` |
| Tampilan | JSON → model → widget; objek `aksi` menentukan tombol | `upgradeStatusMetadata`, `aksi`, `komponen` | `SELECT` saja |

---

## 46. Ringkasan Database Impact

| Event | INSERT | UPDATE | DELETE |
|---|---|---|---|
| Ajukan / Buat | header, item, notif | – | – |
| Review | notif | header | – |
| Ubah jadwal | notif | header | – |
| Serahkan | asset_trx_log, notif | asset_data, header | – |
| Hasil | notif | item, header | – |
| Terima (langsung selesai) | asset_trx_log, notif | asset_data_item (bila berhasil), asset_data, header | – |

Tidak ada lagi event Checklist pengambilan, Selesaikan, dan Kirim ulang (dihapus v1.4). Kolom `checklist_pengambilan` / `checklist_pengembalian` tidak disentuh oleh event mana pun.

---

## 47. Checklist Implementasi

### Database
- [x] Kolom workflow, index, enum log (migration)
- [ ] Selaraskan DBML (index, kolom item, format no_trx; tandai `checklist_*` tidak dipakai)
- [ ] Jalankan `Claude outputs/sql-upgrade-hapus-checklist-review-lama.sql` untuk transaksi lama `menunggu_review` (BELUM dijalankan; uji di staging dulu)
- [ ] Hapus kolom `checklist_pengambilan` / `checklist_pengembalian` dan nilai enum `menunggu_review` (opsional, setelah data lama selesai; butuh persetujuan migrasi)

### Backend
- [x] Jalur A & B, review, jadwal, hasil, serah terima tanpa checklist
- [x] Update atribut `asset_data_item` (kini saat Terima)
- [x] Hapus checklist pengambilan/penerimaan, review akhir (Selesaikan), kirim ulang; rute dan notifikasi terkait (v1.4)
- [x] Test otomatis (5 test, 54 assertion)
- [x] Cek upgrade aktif di Offboarding (v1.1)
- [x] Cek upgrade aktif di Retur (v1.2)
- [ ] Pembatalan / kedaluwarsa
- [ ] Validasi list upgrade dengan pesan yang benar

### Mobile
- [x] List, form (karyawan/admin), detail + aksi (review, ubah jadwal, serahkan, hasil, terima)
- [x] Hapus form checklist, tombol Simpan Checklist Pengambilan / Selesaikan / Kirim Ulang; `terima(id)` tanpa body (v1.4)
- [x] Banner Asset Saya & panel admin
- [x] Test Flutter (5)
- [ ] Bersihkan kunci lokalisasi lama dan teks `pelepasan_info` di `asset.json` (assets/id dan assets/en)
- [ ] Model jadwal sendiri (lepas dari Kerusakan)

### Documentation
- [x] Business Flow, DB, API, Mobile, Mapping, Manual Book, Testing
- [x] Pembaruan v1.4 (checklist dan review akhir dihapus)
- [ ] Screenshot

---

## 48. Changelog

| Version | Date | Changes |
|---|---|---|
| 1.4 | 2026-10-30 | Sinkronisasi dengan kode terbaru: **checklist pengambilan (admin) dan checklist pengembalian/penerimaan (karyawan) dihapus**; aksi admin **Selesaikan** dan **Kirim Ulang ke Teknisi** dihapus; rute `POST upgrade-karyawan/{id}/checklist-pengambilan` (API-007) dan `POST upgrade-karyawan/{id}/selesai` (API-009) dihapus. Penomoran API tidak digeser: kedua nomor tetap ada di tabel §23 sebagai baris "dihapus v1.4" agar rujukan lama tidak bergeser. Endpoint aktif 15 → **13**. Alur baru: Serahkan Aset tanpa checklist; Terima Aset (tanpa body) **langsung menyelesaikan** (atribut diperbarui, asset `terpakai`, log gudang → karyawan, fase `selesai`, `tgl_diterima`). Fase `menunggu_review` tidak dihasilkan lagi (dipertahankan hanya untuk data lama). `aksi` detail kini hanya `review`, `ubah_jadwal`, `pelepasan`, `hasil`, `terima`; field detail `checklist_*` dihapus; pending admin = approval `proses` atau fase `dalam_service`. Notifikasi `upgrade.checklist_pengambilan`, `upgrade.selesai`, `upgrade.kirim_ulang` dihapus; `upgrade.diterima` berakhiran "Upgrade selesai.". Kolom `checklist_pengambilan` / `checklist_pengembalian` tidak diubah tetapi tidak dipakai (ditandai di skema dan ERD). Menulis ulang BR-008/009/011/012 (BR-008 dan BR-012 ditandai dihapus, nomor tidak digeser; menambah BR-014/015), §10–§15, §21, §23–§30, §33–§34, §36–§38, §45.4 (walkthrough 6 langkah, variasi kirim ulang diganti hasil `tidak_dapat_diupgrade`), TC-09 s.d. TC-24, INC-05/06 ditutup, INC-10 s.d. INC-12, RISK-008 s.d. RISK-010, §47. Test: PHPUnit 5 test (54 assertion) dan Flutter 5 test lulus. Skrip `sql-upgrade-hapus-checklist-review-lama.sql` untuk data lama **belum dijalankan** (ketergantungan rilis). Perapihan: checklist Retur (§47) dicentang sesuai RISK-001. |
| 1.3 | 2026-10-22 | Menambah §45.4 *Walkthrough Lengkap dengan Contoh Data* (Jalur A dari ajukan sampai atribut diperbarui: review, checklist pengambilan, serahkan, hasil, terima, selesaikan; variasi Jalur B, tolak, tidak dapat diupgrade, kirim ulang; kasus gagal). Sinkronisasi kode: tabel log ternyata `asset_trx_log` (`id_asset_trx_log`), bukan `asset_log`. |
| 1.2 | 2026-10-12 | RISK-001 ditutup: Retur kini menolak asset dalam upgrade aktif (TC-19, edge case). Sinkronisasi dengan kode 12 Okt 2026: path file mobile mengikuti struktur folder baru (`UI/Modul/Asset/<Fitur>/` berhuruf kapital, model di `Repository/Model/Asset/`). |
| 1.1 | 2026-09-28 | Offboarding kini menolak asset dalam upgrade aktif (RISK-001 sebagian selesai). BR-002 & error 422 menambah blokir Pemeliharaan (PML01). Helper checklist `normalizeChecklist` dipindah ke `Asset_m::normalizeKondisiChecklist` (dipakai bersama PML01, perilaku sama). TC-04, TC-19, edge case, dependency, checklist disesuaikan |
| 1.0 | 2026-09-28 | Initial documentation — disusun dari source lokal API & MOBILE |
