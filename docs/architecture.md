# Arsitektur — Kaimana Data Pipeline

> Tahap 1 (Perencanaan & Desain) · disusun 24 November 2026

## Alur data

```mermaid
flowchart LR
    subgraph Sumber
        A1[BPS WebAPI<br/>tabel dinamis & statis]
        A2[BPS SIMDASI<br/>Yearbook per distrik]
        A3[BMKG<br/>prakiraan cuaca]
    end

    subgraph Ingestion["Ingestion (Python)"]
        B1[ingestion/bps/]
        B2[ingestion/bmkg/]
    end

    subgraph Warehouse["PostgreSQL / Supabase"]
        C1[(raw<br/>JSON & CSV apa adanya)]
        C2[(staging<br/>dibersihkan, tidy)]
        C3[(mart<br/>star schema)]
    end

    D[Orkestrasi<br/>Airflow / Prefect]
    E[Tes kualitas data<br/>dbt tests]
    F[API / Dashboard]
    G[Website Kaimana Insight]

    A1 --> B1
    A2 --> B1
    A3 --> B2
    B1 --> C1
    B2 --> C1
    C1 -->|dbt| C2
    C2 -->|dbt| C3
    C3 --> F --> G
    E -.cek.-> C2
    E -.cek.-> C3
    D -.jadwal.-> B1
    D -.jadwal.-> B2
    D -.jadwal.-> E
```

## Lapisan data

| Lapisan | Isi | Aturan |
|---|---|---|
| **raw** | Respons API mentah (JSON) dan hasil parse awal (CSV), diberi nama per tabel & tahun | Tidak pernah ditimpa atau diubah |
| **staging** | Angka sudah numerik, satuan dikonversi (mis. "ribu" → jiwa), kode wilayah distandarkan, penanda BPS dipisahkan | Satu tabel staging per sumber |
| **mart** | Star schema siap dianalisis | Menjawab langsung pertanyaan bisnis |

## Rancangan awal star schema (mart)

```mermaid
erDiagram
    dim_distrik ||--o{ fact_penduduk : ""
    dim_distrik ||--o{ fact_cuaca : ""
    dim_waktu ||--o{ fact_penduduk : ""
    dim_waktu ||--o{ fact_cuaca : ""
    dim_waktu ||--o{ fact_indikator_kabupaten : ""

    dim_distrik {
        string kode_bps PK
        string kode_kemendagri
        string nama_distrik
        string adm4_titik_cuaca
        string level
    }
    dim_waktu {
        date tanggal PK
        int tahun
        int bulan
    }
    fact_penduduk {
        string kode_bps FK
        int tahun
        float jumlah_penduduk
        float laju_pertumbuhan
        float kepadatan
        float rasio_jk
        string sumber
        string penanda
    }
    fact_cuaca {
        string adm4 FK
        datetime utc_datetime
        datetime analysis_date
        float suhu
        int kelembaban
        float curah_hujan
        string kondisi
        float kecepatan_angin
    }
    fact_indikator_kabupaten {
        string indikator
        int tahun
        float nilai
        string sumber
    }
```

## Jadwal (Tahap 6)

| Job | Jadwal | Isi |
|---|---|---|
| `bmkg_harian` | Setiap hari | Tarik 7 titik cuaca → raw → staging → mart |
| `bps_bulanan` | Setiap bulan | Cek SIMDASI (health check) → tarik tabel baru → dbt run → dbt test |
