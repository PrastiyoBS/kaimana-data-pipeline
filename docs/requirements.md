# Requirements — Kaimana Data Pipeline

> Tahap 1 (Perencanaan & Desain) · disusun 24 November 2026

Pipeline data end-to-end yang mengambil data publik tentang Kabupaten Kaimana (Papua Barat), membersihkan dan menyimpannya di data warehouse, lalu menyajikannya untuk website **Kaimana Insight**.

---

## 1. Pertanyaan bisnis

| # | Pertanyaan | Tingkat wilayah | Sumber utama |
|---|---|---|---|
| 1 | Bagaimana tren jumlah, kepadatan, dan rasio jenis kelamin penduduk di 7 distrik Kaimana dari tahun ke tahun? | Distrik | BPS SIMDASI 3.1.1 |
| 2 | Bagaimana perkembangan pendidikan di Kaimana (rata-rata lama sekolah), dan bagaimana posisinya dibanding kabupaten lain di Papua Barat? | Kabupaten (per distrik menyusul) | BPS tabel dinamis var 343 |
| 3 | Bagaimana tren kemiskinan, pengangguran, pengeluaran per kapita, dan PDRB Kaimana? | Kabupaten | BPS tabel dinamis 179, 270, 269, 240 · SIMDASI 12.1–12.4 |
| 4 | Bagaimana pola cuaca harian di tiap distrik? | Distrik (1 desa per distrik) | BMKG prakiraan cuaca |

---

## 2. Sumber data

### 2.1 BPS WebAPI
- **Base URL:** `https://webapi.bps.go.id/v1/api/`
- **Autentikasi:** API key (App ID) sebagai path parameter `/key/{KEY}`. Disimpan di `.env` sebagai `BPS_API_KEY`, tidak pernah di-commit.
- **Kode domain:** Kaimana `9102`, Papua Barat `9100`.

#### a. Tabel dinamis (domain 9100, per kabupaten/kota)

| var_id | Indikator | Tahun tersedia | Status |
|---|---|---|---|
| 320 | Jumlah penduduk (L/P/Total, proyeksi SP2020) | 2021–2026 | ✅ Dipakai |
| 343 | Rata-rata lama sekolah | 2022–2026 | ✅ Dipakai |
| 179 | Persentase penduduk miskin | 2017–2026 | ✅ Dipakai |
| 270 | Tingkat pengangguran terbuka (TPT) | 2013–2022 (2016 kosong) | ⚠️ Dipakai dengan catatan |
| 269 | TPAK | 2013–2022 (2016 kosong) | ⚠️ Dipakai dengan catatan |
| 240 | Rata-rata pengeluaran per kapita sebulan | 2012–2021 | ⚠️ Berhenti 2021; cek var 341 |
| 341, 345 | Pengeluaran disesuaikan, harapan lama sekolah | belum ditarik | ⏳ |

Alur: daftar periode `list/model/th/domain/9100/var/{var}` → data **satu periode per request** `list/model/data/domain/9100/var/{var}/th/{th}` → saring baris Kaimana.

#### b. Tabel statis
| table_id | Tabel | Kegunaan |
|---|---|---|
| 225 (domain 9100) | Penduduk hasil SP2020 per kab/kota | Pembanding |
| 213 (domain 9100) | Luas wilayah per kab/kota | Menghitung kepadatan |

#### c. SIMDASI (Yearbook / "Kaimana Dalam Angka")
- **Endpoint:** `interoperabilitas/datasource/simdasi/id/{N}/...`
- **Kode wilayah 7 digit:** Papua Barat `9100000`, Kaimana `9102000`.
- ID 27 = daftar kabupaten, ID 28 = daftar distrik, ID 23 = daftar tabel, ID 25 = isi tabel (`wilayah/{w}/tahun/{t}/id_tabel/{id}`).

| Kode | Tabel | Tahun | Status |
|---|---|---|---|
| 3.1.1 | Penduduk, laju pertumbuhan, distribusi, kepadatan, rasio JK per distrik | 2010–2025 | ✅ Ditarik (640 baris) |
| 1.1.1 | Luas daerah dan jumlah pulau per distrik | 2007–2025 | ⏳ |
| 12.1–12.4 | PDRB menurut lapangan usaha | 2015–2025 | ⏳ |
| 4.1.1–4.1.9 | Sekolah, guru, murid per distrik (TK–SMA/SMK, madrasah) | 2016–2025 | ❌ Semua sel `"..."` di API |

### 2.2 BMKG prakiraan cuaca
- **Endpoint:** `https://api.bmkg.go.id/publik/prakiraan-cuaca?adm4={kode_desa}` (tanpa API key)
- **Isi:** 20 titik per 3 jam (±2,5 hari): suhu `t`, kelembaban `hu`, cuaca `weather_desc`, curah hujan `tp`, angin `ws`/`wd`, tutupan awan `tcc`, `analysis_date`.
- **Kode wilayah:** format Kemendagri. Kabupaten Kaimana = `92.08`.
- **Titik pantau (1 desa per distrik, kode terkecil):**

| Distrik | Desa | Kode adm4 |
|---|---|---|
| Kaimana | Kaimana Kota | 92.08.01.1001 |
| Buruway | Kambala | 92.08.02.2001 |
| Teluk Arguni Atas | Furnusu | 92.08.03.2001 |
| Teluk Etna | Rurumo | 92.08.04.2001 |
| Kambrau | Rauna | 92.08.05.2001 |
| Teluk Arguni Bawah | Jawera | 92.08.06.2001 |
| Yamor | Ure | 92.08.07.2001 |

---

## 3. Dimensi wilayah: pemetaan kode BPS ↔ Kemendagri

Urutan kode di kedua sistem **berbeda**, jadi dipetakan berdasarkan nama, bukan rumus. Tabel ini menjadi dasar `dim_distrik`.

| Distrik | Kode BPS (SIMDASI) | Kode Kemendagri (BMKG) |
|---|---|---|
| Kaimana | 9102030 | 92.08.01 |
| Buruway | 9102010 | 92.08.02 |
| Teluk Arguni Atas | 9102020 | 92.08.03 |
| Teluk Etna | 9102040 | 92.08.04 |
| Kambrau | 9102031 | 92.08.05 |
| Teluk Arguni Bawah | 9102021 | 92.08.06 |
| Yamor | 9102041 | 92.08.07 |
| *(Kabupaten)* | 9102000 | 92.08 |

---

## 4. Frekuensi update

| Sumber | Frekuensi rilis | Jadwal pipeline |
|---|---|---|
| BMKG prakiraan cuaca | Diperbarui beberapa kali sehari | **Harian** |
| BPS tabel dinamis | Tahunan | Bulanan (cek data baru) |
| BPS SIMDASI | Tahunan (umumnya Februari–Maret) | Bulanan (cek data baru) |
| BPS tabel statis | Jarang | Manual / sekali tarik |

---

## 5. Keterbatasan data

1. **BMKG tidak menyimpan riwayat.** API hanya memberi prakiraan ±3 hari. Riwayat cuaca harus dikumpulkan sendiri oleh pipeline setiap hari, mulai sedini mungkin. Kunci unik: `adm4` + `utc_datetime` + `analysis_date`.
2. **Pendidikan per distrik kosong di SIMDASI.** Semua sel tabel 4.1.x berisi `"..."`. Sumber aslinya Dapodik (Kemendikdasmen); akan dicoba di Tahap 2. Sementara dipakai var 343 tingkat kabupaten.
3. **Total kabupaten SIMDASI 2025 salah.** Jumlah penduduk kabupaten tercatat 6,6 ribu, padahal jumlah 7 distrik 65,6 ribu (selisih −89,9%). Total kabupaten dihitung ulang dari jumlah distrik; nilai asli tetap disimpan dengan penanda.
4. **Patahan seri penduduk 2018→2019.** Distrik Kaimana naik 16%, Yamor turun 23%. Kemungkinan ada perubahan sumber atau metode; tren lintas 2019 diberi catatan.
5. **Dua sumber jumlah penduduk tidak boleh dicampur.** SIMDASI 3.1.1 memakai data Dukcapil (±65,6 ribu, 2025), var 320 memakai proyeksi BPS (67.795, 2025).
6. **Anomali murid SMP 2015** (var 189): tercatat 16, seharusnya ±2.760.
7. **SIMDASI bisa mati sementara.** Pada 18 Nov 2026 server mengembalikan HTML "500 CDbConnection failed" dibungkus respons HTTP 200; pulih 24 Nov 2026. Selalu jalankan `cek_simdasi.py` dan validasi isi respons.
8. **Data per kabupaten di domain 9100 tidak dirinci per distrik.**
9. **Pemekaran Papua Barat Daya:** beberapa variabel domain 9100 mencatat data sebelum 2023 masih termasuk Papua Barat Daya. Kaimana tetap di Papua Barat.

---

## 6. Aturan kualitas data (rancangan awal untuk Tahap 5)

| Aturan | Ambang |
|---|---|
| Jumlah distrik vs total kabupaten | Selisih > 1% → peringatan |
| Perubahan antar tahun per distrik | > 50% → peringatan |
| Nilai negatif pada jumlah (penduduk, sekolah, guru, murid) | Tidak boleh |
| Kode wilayah | Wajib ada di `dim_distrik` |
| Respons API | Harus JSON valid, bukan HTML; tabel 404 dilewati dan dicatat |
| Tabel yang semua selnya kosong | Peringatan, jangan disimpan sebagai data valid |

---

## 7. Keamanan

- API key hanya di `.env` (di-ignore Git). `.env.example` berisi placeholder.
- Jangan menempel API key di chat, screenshot, atau log.
- Folder `data/raw/` tidak di-commit.
