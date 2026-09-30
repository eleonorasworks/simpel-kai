# SIMPEL PKL

**Sistem Manajemen Peserta PKL / Magang — Unit Sistem Informasi, PT Kereta Api Indonesia (Persero) Daop 6 Yogyakarta**

SIMPEL PKL adalah aplikasi web satu berkas (*single-file*) untuk membantu unit mengelola peserta Praktik Kerja Lapangan (PKL) dan magang: dari pendataan pemohon, kelengkapan berkas, absensi, penilaian, sampai penerbitan sertifikat. Aplikasi ditulis dengan HTML, CSS, dan JavaScript murni tanpa framework maupun proses *build*.

> **Status:** prototipe / demo. Data awal berisi nama peserta fiktif. Lihat bagian [Keterbatasan & Catatan Keamanan](#keterbatasan--catatan-keamanan) sebelum dipakai dengan data sungguhan.

---

## Daftar Isi

1. [Fitur](#fitur)
2. [Teknologi](#teknologi)
3. [Cara Menjalankan](#cara-menjalankan)
4. [Penyimpanan Data & Setup Supabase](#penyimpanan-data--setup-supabase)
5. [Hak Akses](#hak-akses)
6. [Struktur Data](#struktur-data)
7. [Impor & Ekspor](#impor--ekspor)
8. [Struktur Kode](#struktur-kode)
9. [Keterbatasan & Catatan Keamanan](#keterbatasan--catatan-keamanan)
10. [Rencana Pengembangan](#rencana-pengembangan)

---

## Fitur

| Menu | Kegunaan |
|---|---|
| **Dashboard** | Ringkasan peserta aktif, sedang berjalan, menunggu mulai, jumlah instansi; grafik sebaran per instansi dan per bulan mulai; aktivitas terakhir. |
| **Data Peserta** | Tambah, ubah, lihat detail, nonaktifkan, dan hapus peserta. Dilengkapi pencarian, filter, pengurutan kolom, dan paginasi. |
| **Instansi** | Rekap sekolah/kampus asal peserta beserta jenjang dan jurusannya. |
| **Berkas** | Checklist surat pengantar, CV, dan surat balasan per peserta (*Menunggu / Kurang / Lengkap*). |
| **Penilaian** | Nilai per aspek (Disiplin, Kerja sama, Keterampilan, Inisiatif), nilai akhir otomatis, predikat, dan catatan pembimbing. |
| **Absensi** | Logbook harian (Hadir / Izin / Sakit / Alfa) beserta kegiatan, rekap per bulan, dan persentase kehadiran. |
| **Sertifikat** | Unduh sertifikat peserta (A4 lanskap) dalam bentuk HTML yang siap dicetak atau disimpan sebagai PDF. |
| **Kalender** | Linimasa periode PKL seluruh peserta aktif, lengkap dengan penanda hari ini. |
| **Laporan** | Statistik status, jenjang pendidikan, dan jurusan terbanyak; dapat dicetak. |
| **Riwayat** | Log perubahan data (60 entri terakhir). *Khusus Admin.* |
| **Pengguna** | Manajemen akun dan peran. *Khusus Admin.* |
| **Pengaturan** | Cadangkan/pulihkan data (JSON) dan reset ke data awal. *Khusus Admin.* |

Fitur pendukung:

- **Notifikasi** untuk peserta yang periode PKL-nya berakhir dalam 14 hari.
- **Tema terang/gelap** yang tersimpan di browser.
- **Tampilan responsif** dengan sidebar yang dapat dibuka-tutup di layar kecil.
- **Indikator status penyimpanan** di *topbar*: `☁ Online` (tersinkron ke Supabase) atau `💾 Lokal` (hanya di browser).

### Status peserta

Status dihitung otomatis dari tanggal periode PKL:

| Status | Kondisi |
|---|---|
| Menunggu mulai | Tanggal mulai belum tiba |
| Berjalan | Hari ini berada di antara tanggal mulai dan selesai |
| Selesai | Tanggal selesai sudah lewat |
| Nonaktif | Peserta dinonaktifkan oleh Admin |

### Predikat nilai

| Nilai akhir | Predikat |
|---|---|
| ≥ 85 | Sangat Baik |
| 70 – 84 | Baik |
| 55 – 69 | Cukup |
| < 55 | Kurang |

Nilai akhir adalah rata-rata dari aspek yang sudah diisi (nilai 0–100).

---

## Teknologi

- HTML5, CSS3 (variabel CSS, mode terang/gelap), dan JavaScript ES6+ tanpa dependensi.
- Font **Plus Jakarta Sans** dari Google Fonts.
- **Supabase** (REST API / PostgREST) sebagai penyimpanan bersama, dengan **localStorage** sebagai cache dan cadangan.

---

## Cara Menjalankan

### Lokal

Tidak perlu instalasi. Cukup buka berkas `simpel-pkl-kai.html` di browser modern (Chrome, Edge, Firefox, atau Safari).

Jika ingin disajikan lewat server lokal:

```bash
# Python
python3 -m http.server 8080

# atau Node.js
npx serve .
```

Lalu buka `http://localhost:8080`.

### Hosting

Karena hanya satu berkas statis, aplikasi dapat di-*deploy* ke layanan hosting statis apa pun (GitHub Pages, Netlify, Vercel, Cloudflare Pages, atau server web internal). Ganti nama berkas menjadi `index.html` agar terbuka langsung pada alamat utama.

### Login

Halaman login bersifat **demo**: isi email dan kata sandi dengan apa saja, pilih peran (**Admin** atau **Staf**), lalu tekan *Masuk*. Lihat [catatan keamanan](#keterbatasan--catatan-keamanan).

---

## Penyimpanan Data & Setup Supabase

Alur penyimpanan data:

1. Saat dibuka, aplikasi memuat data dari **localStorage** agar tampilan langsung terisi.
2. Aplikasi lalu mencoba mengambil data dari **Supabase**. Jika berhasil, data cloud menggantikan data lokal dan status berubah menjadi `☁ Online`.
3. Setiap perubahan disimpan ke localStorage dan dikirim ke Supabase.
4. Selama online, aplikasi menyegarkan data dari cloud setiap **15 detik**.
5. Jika Supabase tidak terjangkau, aplikasi tetap berjalan memakai data lokal (`💾 Lokal`).

Seluruh data aplikasi disimpan sebagai **satu dokumen JSON** pada satu baris tabel `app_state` dengan `id = 'main'`.

### Membuat tabel

Jalankan di *SQL Editor* Supabase:

```sql
create table if not exists public.app_state (
  id         text primary key,
  data       jsonb not null,
  updated_at timestamptz not null default now()
);

alter table public.app_state enable row level security;
```

Aplikasi ini tidak memakai Supabase Auth, sehingga aksesnya lewat peran `anon`. Untuk **demo/pengembangan saja**, kebijakan berikut memberi akses baca-tulis penuh kepada siapa pun yang memegang *publishable key*:

```sql
create policy "demo anon read"   on public.app_state for select to anon using (true);
create policy "demo anon insert" on public.app_state for insert to anon with check (true);
create policy "demo anon update" on public.app_state for update to anon using (true) with check (true);
```

> ⚠️ Jangan gunakan kebijakan di atas untuk data peserta sungguhan. Lihat bagian keamanan di bawah.

### Menghubungkan ke proyek Supabase sendiri

Buka berkas HTML, cari dua konstanta berikut, lalu isi dengan nilai dari **Project Settings → API**:

```js
const SUPA_URL = "https://<project-ref>.supabase.co";
const SUPA_KEY = "<publishable / anon key>";
```

Gunakan hanya *publishable (anon) key*. **Jangan pernah** menaruh `service_role` / *secret key* di berkas ini karena kode berjalan di sisi browser dan dapat dilihat siapa saja.

### Bekerja tanpa Supabase

Kosongkan nilai `SUPA_URL` atau biarkan koneksi gagal: aplikasi otomatis berjalan dalam mode lokal. Gunakan **Pengaturan → Unduh cadangan (JSON)** secara berkala karena data lokal hanya ada di browser tersebut.

---

## Hak Akses

| Kemampuan | Admin | Staf |
|---|:---:|:---:|
| Melihat dashboard, peserta, instansi, kalender, laporan | ✅ | ✅ |
| Menambah peserta, mengelola berkas/nilai/absensi, unduh sertifikat | ✅ | ✅ |
| Mengubah, menonaktifkan, dan menghapus peserta | ✅ | ❌ |
| Riwayat aktivitas | ✅ | ❌ |
| Manajemen pengguna | ✅ | ❌ |
| Pengaturan (cadangan, pemulihan, reset) | ✅ | ❌ |

---

## Struktur Data

Setiap peserta disimpan sebagai objek berikut:

```jsonc
{
  "id": "PKL-001",
  "nama": "Andika Pratama Putra",
  "jenjang": "SMA/SMK",            // SMA/SMK | D3 | D4/S1 | S2
  "instansi": "SMK Negeri 2 Yogyakarta",
  "jurusan": "Sistem Informasi Jaringan dan Aplikasi",
  "lokasi": "DAOP 6 Yogyakarta",
  "unit": "Sistem Informasi",
  "pic": "Nama PIC Unit",
  "tglDaftar": "2026-07-28",
  "tenggat": "2026-08-04",
  "mulai": "2026-08-28",
  "selesai": "2027-02-25",
  "statusUnit": "Disetujui",       // Menunggu | Disetujui
  "aktif": true,
  "berkas": { "pengantar": "Lengkap", "cv": "Kurang", "balasan": "Menunggu" },
  "nilai":  { "disiplin": 90, "kerjasama": 85, "keterampilan": 80, "inisiatif": 88 },
  "catatan": "…",
  "absen":  [ { "t": "2026-09-01", "s": "H", "k": "Membuat halaman login" } ]
}
```

Struktur dokumen penyimpanan secara keseluruhan:

```jsonc
{
  "peserta": [ /* daftar peserta */ ],
  "log":     [ /* riwayat perubahan, maksimal 60 entri */ ],
  "profile": { "nama": "…", "email": "…" },
  "users":   [ { "id": "U-001", "nama": "…", "email": "…", "peran": "Admin" } ]
}
```

Kode absensi (`s`): `H` Hadir, `I` Izin, `S` Sakit, `A` Alfa.

---

## Impor & Ekspor

| Aksi | Lokasi | Format |
|---|---|---|
| Ekspor data peserta | Data Peserta → Unduh CSV | CSV, pemisah `;`, hasil mengikuti filter yang aktif |
| Template impor | Data Peserta → Template CSV | CSV, pemisah `;` |
| Impor peserta | Data Peserta → Impor CSV | CSV, pemisah `;` atau `,` |
| Cadangan penuh | Pengaturan → Unduh cadangan | JSON |
| Pemulihan | Pengaturan → Pulihkan dari berkas | JSON hasil cadangan |
| Sertifikat | Berkas/Penilaian/Sertifikat → Unduh sertifikat | HTML (cetak atau simpan sebagai PDF) |

Kolom CSV impor: `Nama; Jenjang; Instansi; Jurusan; Unit; PIC; Periode Awal; Periode Akhir` dengan tanggal berformat `YYYY-MM-DD`. Baris tanpa nama dilewati.

Contoh:

```csv
sep=;
"Nama";"Jenjang";"Instansi";"Jurusan";"Unit";"PIC";"Periode Awal";"Periode Akhir"
"Contoh Nama";"SMA/SMK";"SMK Negeri 2 Yogyakarta";"Rekayasa Perangkat Lunak";"Sistem Informasi";"Nama PIC";"2026-10-01";"2027-01-31"
```

---

## Struktur Kode

Seluruh aplikasi berada dalam satu berkas dengan urutan kira-kira sebagai berikut:

| Bagian | Isi |
|---|---|
| `<style>` | Desain, variabel warna, tema terang/gelap, tata letak responsif |
| Markup | Sidebar navigasi, *topbar*, kontainer tampilan, modal, toast, login |
| `SEED` | Data awal peserta (fiktif) |
| Storage | `cloudLoad`, `cloudSave`, `load`, `save`, `initData`, `logIt` |
| Render | `render`, `renderLogin`, `view*` (dashboard, peserta, instansi, laporan, riwayat, pengguna, pengaturan, kalender, kelola) |
| Form & modal | `openForm`, `openDetail`, `openManage`, `openUserForm`, `openProfileForm`, `confirmBox` |
| Utilitas | `saveFile`, `exportCSV`, `importCSV`, `exportBackup`, `restoreBackup`, `certHTML`, `nilaiAkhir`, `predikat` |

Perubahan yang paling sering dibutuhkan:

- **Mengganti data awal:** ubah array `SEED`. Data ini hanya dipakai pada pengguna baru atau setelah *Reset data*.
- **Mengganti aspek penilaian atau daftar berkas:** ubah konstanta `NILAI` dan `BERKAS`.
- **Mengganti warna/identitas:** ubah variabel CSS di blok `:root`.
- **Mengganti isi sertifikat:** ubah fungsi `certHTML`.

---

## Keterbatasan & Catatan Keamanan

Bagian ini penting dibaca sebelum aplikasi dipakai untuk operasional.

1. **Login belum sungguhan.** Tidak ada verifikasi email atau kata sandi, dan peran dipilih sendiri di halaman login. Siapa pun yang bisa membuka aplikasi dapat masuk sebagai Admin. Pembatasan hak akses hanya berlaku di antarmuka, bukan di server.
2. **Akses database bergantung pada RLS.** *Publishable key* memang aman untuk dilihat publik, tetapi hanya jika *Row Level Security* dikonfigurasi dengan benar. Dengan kebijakan demo di atas, siapa pun yang memiliki URL dan key (terlihat di kode sumber) dapat membaca dan menimpa seluruh data peserta.
3. **Data pribadi.** Data peserta (nama, sekolah, jurusan, penilaian) tergolong data pribadi. Untuk data sungguhan, gunakan autentikasi (misalnya Supabase Auth) dan kebijakan RLS per pengguna/peran.
4. **Penulisan bersifat *last-write-wins*.** Seluruh data disimpan sebagai satu dokumen. Jika dua orang mengedit bersamaan, penyimpanan terakhir menimpa yang sebelumnya.
5. **Reset data menimpa cloud.** Tombol *Reset ke data awal* juga menulis data awal ke Supabase, bukan hanya di browser.
6. **Cadangan.** Karena tidak ada versi/riwayat di sisi database, lakukan unduh cadangan JSON secara berkala.
7. **Teks pada halaman Pengaturan** masih menyebut data tersimpan "di browser perangkat ini saja", padahal saat terhubung Supabase data juga tersimpan di cloud.
8. **Ketergantungan eksternal.** Font memuat dari Google Fonts; tanpa internet, aplikasi tetap berjalan dengan font bawaan sistem.

---

## Rencana Pengembangan

- Autentikasi sungguhan (Supabase Auth) dengan peran Admin/Staf yang ditegakkan di sisi server.
- Memecah dokumen tunggal menjadi tabel relasional (`peserta`, `absensi`, `log`, `users`) agar aman untuk penyuntingan bersamaan.
- Unggah dan penyimpanan berkas (surat pengantar, CV, surat balasan) melalui Supabase Storage.
- Ekspor sertifikat dan laporan langsung ke PDF.
- Pengingat otomatis lewat email untuk periode PKL yang hampir berakhir.

---

## Lisensi

Belum ditentukan. Sesuaikan dengan kebijakan unit/instansi sebelum dipublikasikan.
