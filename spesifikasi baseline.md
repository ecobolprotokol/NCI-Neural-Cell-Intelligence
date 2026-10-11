# NCI (Neural Cell Intelligence): Spesifikasi Teknis Lengkap v2.0.0

**Status:** Spesifikasi normatif tunggal untuk seluruh proyek, dari format data hingga produk siap deployment. Dokumen ini menggantikan Engineering Contract v1.0.0 dan tidak memuat fase, roadmap, atau rencana bertahap.

---

## 0. Pendahuluan dan Konvensi

### 0.1 Konvensi Bahasa Normatif

Kata **WAJIB**, **HARUS**, **TIDAK BOLEH**, **SEBAIKNYA**, dan **BOLEH** dalam huruf kapital ditafsirkan sesuai RFC 2119. Setiap klausul diberi pengidentifikasi `REQ-<bagian>-<nomor>` agar dapat dirujuk oleh uji, tinjauan kode, dan perubahan spesifikasi.

Ukuran dalam dokumen ini memakai satuan biner (KiB = 1024 byte) kecuali dinyatakan lain. Urutan byte di seluruh format adalah little-endian.

### 0.2 Cakupan Dokumen

Dokumen ini mendefinisikan:

1. Format berkas `.ncl` pada tingkat byte.
2. Model data (Cell, kolom, tensor, graph, memori, revisi).
3. Model komputasi dan algoritma eksekusi.
4. Program tunggal `nci` yang mencakup pembuatan model, inspeksi, verifikasi, inferensi, dan layanan API.
5. Keamanan, determinisme, pengujian, dan kriteria penerimaan.

Setiap hal yang tidak didefinisikan di sini bukan bagian dari produk. Perilaku yang tidak dispesifikasikan TIDAK BOLEH ditambahkan secara diam-diam, dan harus diajukan sebagai amandemen (Bagian 38).

### 0.3 Keputusan Kunci

Keputusan berikut menutup ambiguitas pada spesifikasi sebelumnya:

- **D-01 Dua identitas.** Setiap Cell logis memiliki `CellId` yang stabil lintas kuantisasi. Setiap representasi fisik memiliki `RepresentationId` sendiri. Keduanya memakai BLAKE3-256 (Bagian 9).
- **D-02 Format append-only dengan dua slot superblock.** Komit ditulis dengan menulis slot superblock yang tidak aktif setelah seluruh data terkomit ke disk. Ini memberi atomisitas tanpa berkas sementara ganda (Bagian 15).
- **D-03 Satu binary, dua lapisan.** Binary `nci` boleh memuat kode konverter, tetapi jalur runtime (`run`, `serve`) tidak boleh memanggil parser format eksternal dan tidak boleh memiliki konverter dalam graf dependensinya. Hal ini diverifikasi di CI (Bagian 6.4 dan 26).
- **D-04 Tidak ada Python atau PyTorch di runtime maupun konverter.** Parser PyTorch ditulis di Rust dengan unpickler terbatas (allowlist), bukan dengan memanggil `torch.load` (Bagian 26.2).
- **D-05 Pembelajaran inkremental v1 bukan pelatihan gradien.** NCI tidak melakukan backpropagation. Pembelajaran berarti penggantian Cell oleh Cell yang diimpor dari luar, konsolidasi memori, dan pembaruan Routing melalui revisi baru (Bagian 23).
- **D-06 Klaim kapasitas 1 TB di RAM 16 GB dinyatakan sebagai kriteria uji terukur**, bukan jaminan latensi (Bagian 35).
- **D-07 Himpunan operator model dibatasi** pada keluarga transformer decoder-only dengan MoE opsional (Bagian 21). Arsitektur lain memerlukan amandemen.

---

## 1. Ruang Lingkup dan Non-Goal

### 1.1 Ruang Lingkup v1

- REQ-1-01: NCI v1 MUST mendukung inferensi teks autoregresif untuk model decoder-only transformer dengan RoPE, GQA, RMSNorm, SwiGLU, dan opsional Mixture-of-Experts top-k.
- REQ-1-02: NCI v1 MUST menjalankan model dari berkas `.ncl` lokal atau dari berkas yang sudah dikonversi, tanpa koneksi jaringan.
- REQ-1-03: NCI v1 MUST menyediakan penyimpanan memori persisten lintas sesi melalui Cell memori.
- REQ-1-04: NCI v1 MUST menyediakan API lokal berbasis HTTP/JSON dan API Rust sebagai library.

### 1.2 Non-Goal v1

- Pelatihan model (forward dan backward pass, optimizer).
- Multimodal (gambar, audio, video). Tipe kolom untuk media dicadangkan, tetapi tidak diimplementasikan.
- Distribusi terdesentralisasi antar node.
- Antarmuka grafis. Produk hanya berupa CLI dan API.
- Dukungan SentencePiece atau tokenizer selain byte-level BPE (Bagian 21.5).

---

## 2. Istilah dan Definisi

| Istilah | Definisi |
|---|---|
| **Cell** | Unit logis terkecil yang berisi metadata dan payload tak-berubah. |
| **Logical Cell** | Identitas Cell yang stabil terhadap representasi kuantisasi. |
| **Representation** | Varian fisik dari Logical Cell (misalnya FP16 atau INT4). |
| **Payload** | Byte mentah (tidak terkompresi) yang dihash untuk identitas. |
| **Block** | Unit penyimpanan fisik terkompresi yang memuat sebagian atau seluruh payload. |
| **Kolom** | Array homogen berisi satu atribut untuk seluruh baris (Cell, edge, memori, dll). |
| **Region** | Bagian berkas dengan header dan tipe tetap. |
| **Komit** | Operasi atomik yang menghasilkan revisi baru. |
| **Revisi** | Snapshot tak-berubah dari keadaan model, dengan parent yang membentuk DAG. |
| **Budget** | Batas keras untuk satu eksekusi (kedalaman, Cell, byte, waktu). |
| **Routing** | Proses memilih subset Cell untuk satu langkah komputasi. |
| **Working memory** | Keadaan sesi aktif yang dibatasi ukurannya. |
| **Active set** | Himpunan Cell yang dimuat ke RAM atau VRAM pada satu waktu. |
| **Backend** | Implementasi komputasi untuk perangkat tertentu (CPU, CUDA, Metal, Vulkan). |

---

## 3. Persyaratan Single Binary

- REQ-3-01: Produk akhir MUST berupa satu berkas eksekusi bernama `nci` untuk setiap target platform. Tidak ada komponen tambahan yang wajib diinstal (tidak ada Python, Node, JVM, atau runtime bahasa lain).
- REQ-3-02: Satu berkas `nci` MUST mampu melakukan seluruh operasi: konversi, pembuatan revisi, verifikasi, inspeksi, inferensi, layanan API, pembangkitan konfigurasi, dan self-test.
- REQ-3-03: Aset yang diperlukan runtime (tabel kuantisasi, vektor uji self-test, skema bawaan, sertifikat akar trust store bawaan jika ada) MUST dikompilasi ke dalam binary menggunakan `include_bytes!` atau setara.
- REQ-3-04: Pustaka dinamis sistem yang WAJIB ada pada platform MUST diminimalkan. Target Linux MUST dibangun dengan musl agar statis penuh. Target macOS MUST hanya bergantung pada `libSystem`. Target Windows MUST hanya bergantung pada runtime Windows bawaan.
- REQ-3-05: Dukungan GPU (CUDA, Metal, Vulkan) MUST dimuat secara dinamis saat runtime (`dlopen`/`LoadLibrary`) dan MUST gagal dengan aman (fallback ke CPU) jika pustaka driver tidak ditemukan. Ketiadaan GPU MUST NOT mencegah binary berjalan.
- REQ-3-06: Binary MUST menyertakan versi, hash build (commit Git), dan target triple melalui perintah `nci version`.
- REQ-3-07: Binary MUST dapat berjalan tanpa akses jaringan. Tidak ada panggilan jaringan yang dilakukan secara implisit. Pengunduhan model dari internet MUST NOT menjadi fitur bawaan v1.
- REQ-3-08: Ukuran binary rilis SEBAIKNYA tidak melebihi 60 MiB untuk target CPU saja, dengan fitur GPU dinonaktifkan secara default. Angka ini adalah tujuan pengukuran dan bukan syarat fungsional.

---

## 4. Prinsip Arsitektur

Dua belas prinsip berikut bersifat normatif. Prinsip tambahan berawalan NCI-P13 sampai NCI-P15 melengkapi daftar.

- **NCI-P01 Universal Cell.** Semua informasi model (bobot, metadata, routing, memori, graph, provenans) direpresentasikan sebagai Cell.
- **NCI-P02 Columnar.** Data yang dapat diakses atau diubah secara independen disimpan dalam format kolom. Penyimpanan baris campuran dilarang kecuali untuk header region tetap.
- **NCI-P03 Content addressing.** Identitas diturunkan dari isi payload tidak terkompresi.
- **NCI-P04 Immutability.** Cell yang sudah terkomit tidak boleh diubah. Perubahan menghasilkan Cell baru dengan identitas baru.
- **NCI-P05 Selective activation.** Runtime hanya memuat Cell yang diperlukan.
- **NCI-P06 Dynamic computation.** Jalur eksekusi ditentukan oleh input, routing, dan budget.
- **NCI-P07 Persistent memory.** Memori bertahan lintas sesi melalui revisi.
- **NCI-P08 Streaming-first.** Konversi, hashing, dan penulisan memproses data secara bertahap.
- **NCI-P09 Zero format coupling pada runtime.** Jalur runtime hanya memahami `.ncl`.
- **NCI-P10 Hardware independence.** Representasi hardware-specific boleh ada, tetapi tidak boleh wajib untuk kebenaran.
- **NCI-P11 Incremental evolution.** Perubahan terjadi pada tingkat Cell atau tabel metadata per revisi.
- **NCI-P12 Data movement first.** Metrik utama optimasi adalah byte yang dipindahkan per komputasi yang berguna.
- **NCI-P13 Explicit failure.** Tidak ada kegagalan senyap. Setiap kegagalan menghasilkan kode error dari Bagian 30.
- **NCI-P14 Bounded resources.** Setiap alokasi yang bergantung pada input MUST memiliki batas eksplisit.
- **NCI-P15 Reproducible artifacts.** Build dan berkas `.ncl` yang sama dari input yang sama MUST menghasilkan byte yang identik.

---

## 5. Arsitektur Sistem

### 5.1 Lapisan Logis

```
┌────────────────────────────────────────────────────────────┐
│ Antarmuka: nci CLI  │  HTTP/JSON API  │  Rust library API  │
├────────────────────────────────────────────────────────────┤
│ Runtime: sesi, decoding, routing, budget, memori           │
├────────────────────────────────────────────────────────────┤
│ Backend: cpu (wajib), cuda / metal / vulkan (opsional)     │
├────────────────────────────────────────────────────────────┤
│ Model data: Cell, Kolom, Graph, Routing, Memori, Revisi    │
├────────────────────────────────────────────────────────────┤
│ Penyimpanan: NCL reader/writer, indeks, cache, komit       │
├────────────────────────────────────────────────────────────┤
│ Dasar: kompresi (zstd), kuantisasi, hash (BLAKE3), CRC32C,  │
│        keamanan (Ed25519, XChaCha20-Poly1305, Argon2id)     │
└────────────────────────────────────────────────────────────┘
          ▲                                  ▲
   Konverter (hanya cmd `convert`)    Verifikasi (`check`, `inspect`)
```

### 5.2 Aliran Data Inferensi

1. Runtime membuka berkas `.ncl`, memverifikasi superblock dan manifest, serta memuat indeks.
2. Runtime memilih revisi target (default: revisi terbaru yang valid).
3. Tokenizer dimuat dari Cell TOKENIZER.
4. Prompt diubah menjadi token ID.
5. Untuk setiap langkah, Routing memilih Cell, storage mengambil payload yang diperlukan melalui cache, backend mengeksekusi operator, dan hasil dievaluasi terhadap kebijakan.
6. Token dipilih melalui sampling, lalu dikirim ke pemanggil.
7. Setelah sesi berakhir, memori sesi dapat dikonsolidasikan dan dikomit sebagai revisi baru.

---

## 6. Struktur Repositori dan Dependensi

### 6.1 Workspace

REQ-6-01: Repositori MUST berupa satu Cargo workspace dengan struktur berikut:

```
nci/
├── Cargo.toml                 (workspace)
├── rust-toolchain.toml        (versi stable terkunci)
├── deny.toml                  (cargo-deny: lisensi, advisory, sumber)
├── crates/
│   ├── nci-core               (tipe dasar, error, varint, canonical encoding, hash)
│   ├── nci-observability      (registry metrik, logging terstruktur)
│   ├── nci-cell               (model Cell, skema, atribut)
│   ├── nci-compress           (zstd, registry codec)
│   ├── nci-quant              (FP8/INT8/INT4/INT3, dequantisasi)
│   ├── nci-integrity          (CRC32C, validasi hash, quarantine)
│   ├── nci-security           (Ed25519, trust store, XChaCha20-Poly1305, Argon2id)
│   ├── nci-ncl                (format NCL: pembaca dan penulis tingkat byte)
│   ├── nci-storage            (komit, lock, indeks, cache, payload resolver)
│   ├── nci-graph              (edge kolom, traversal)
│   ├── nci-routing            (scorer, seleksi Cell, kebijakan eksekusi)
│   ├── nci-memory             (tabel memori, retrieval, konsolidasi)
│   ├── nci-revision           (DAG revisi, merge check)
│   ├── nci-backend-api        (trait ComputeBackend)
│   ├── nci-backend-cpu        (backend CPU referensi)
│   ├── nci-backend-gpu        (loader dinamis CUDA, Metal, Vulkan)
│   ├── nci-runtime            (sesi, decoding loop, budget)
│   ├── nci-config             (konfigurasi, precedence)
│   ├── nci-api                (server HTTP, OpenAPI)
│   ├── nci-converter          (orkestrasi konversi, streaming)
│   ├── nci-import-pytorch     (parser zip/pickle terbatas)
│   ├── nci-import-safetensors
│   ├── nci-import-gguf
│   ├── nci-testkit            (model sintetis deterministik, helper uji)
│   └── nci-cli                (binary `nci`)
├── conformance/               (vektor uji, berkas referensi, checksum)
├── fuzz/                      (target cargo-fuzz)
├── benches/                   (benchmark)
└── ci/                        (skrip verifikasi)
```

### 6.2 Aturan Dependensi

REQ-6-02: Graf dependensi MUST mengikuti lapisan berikut. Dependensi hanya boleh mengarah ke lapisan yang lebih rendah atau setara yang tidak menciptakan siklus.

- **L0:** `nci-core`, `nci-observability`
- **L1:** `nci-cell`, `nci-compress`, `nci-quant`, `nci-integrity`, `nci-security`, `nci-backend-api`, `nci-config`
- **L2:** `nci-ncl`
- **L3:** `nci-storage`, `nci-graph`, `nci-revision`
- **L4:** `nci-routing`, `nci-memory`, `nci-backend-cpu`, `nci-backend-gpu`
- **L5:** `nci-runtime`
- **L6:** `nci-api`
- **Konverter (terpisah):** `nci-import-*` (hanya bergantung pada `nci-core`), `nci-converter` (bergantung pada `nci-core`, `nci-cell`, `nci-quant`, `nci-compress`, `nci-storage`, dan `nci-import-*`)
- **Composition root:** `nci-cli` (boleh bergantung pada semua)

REQ-6-03: `nci-graph` MUST NOT bergantung pada `nci-storage`. Akses data graph MUST melalui trait `NclColumnSource` yang didefinisikan di `nci-ncl`.

REQ-6-04: `nci-runtime` MUST NOT bergantung, baik langsung maupun transitif, pada crate `nci-import-*` atau `nci-converter`. Ini diverifikasi dengan `cargo tree -p nci-runtime` di CI. Kegagalan verifikasi memblokir merge.

REQ-6-05: `nci-observability` MUST bergantung hanya pada `nci-core`. Crate lain MUST boleh bergantung padanya.

### 6.3 Konvensi Kode

- REQ-6-06: Crate `nci-core`, `nci-ncl`, `nci-integrity`, `nci-security`, dan `nci-quant` MUST memakai `#![forbid(unsafe_code)]` kecuali modul yang disetujui secara eksplisit (SIMD, mmap).
- REQ-6-07: Setiap blok `unsafe` yang diizinkan MUST memiliki komentar `// SAFETY:` yang menjelaskan invarian dan MUST tercakup oleh uji.
- REQ-6-08: Setiap crate MUST memiliki lint `clippy::pedantic` dengan pengecualian yang didokumentasikan.
- REQ-6-09: Parsing input eksternal (format NCL, konverter) MUST tidak pernah melakukan panic. Setiap kondisi tak valid MUST menghasilkan error.

### 6.4 Batas Runtime dan Konverter Dalam Satu Binary

REQ-6-10: Binary `nci` MUST menghubungkan kode konverter, tetapi jalur perintah `run`, `serve`, `check`, dan `inspect` MUST hanya memanggil fungsi dari `nci-runtime`, `nci-ncl`, dan `nci-storage`.

REQ-6-11: Perintah `run` dan `serve` MUST menolak berkas yang bukan `.ncl` dengan `NCI-E-FORMAT` tanpa mencoba membaca isinya.

---

## 7. Toolchain, Platform, dan Build

- REQ-7-01: Bahasa implementasi adalah Rust. Toolchain MUST dikunci melalui `rust-toolchain.toml` pada channel stable dengan versi eksplisit.
- REQ-7-02: MSRV MUST dinyatakan di `Cargo.toml` dan MUST diuji di CI. Menaikkan MSRV adalah perubahan yang memerlukan catatan dalam log perubahan.
- REQ-7-03: Target rilis wajib: `x86_64-unknown-linux-musl`, `aarch64-unknown-linux-musl`, `aarch64-apple-darwin`, `x86_64-apple-darwin`, dan `x86_64-pc-windows-msvc`.
- REQ-7-04: Target `riscv64gc-unknown-linux-musl` SEBAIKNYA didukung sebagai target tambahan.
- REQ-7-05: Build MUST reproducible. Dengan `SOURCE_DATE_EPOCH`, flag `--remap-path-prefix`, dan `Cargo.lock` terkomit, dua build dari commit yang sama MUST menghasilkan binary yang bit-identik pada target yang sama.
- REQ-7-06: Dependensi pihak ketiga MUST di-pin di `Cargo.lock`. Setiap pembaruan MUST melewati `cargo-deny` dan `cargo-audit`.
- REQ-7-07: Build fitur default MUST mencakup `backend-cpu`, `converter`, `security`, dan `api`. Fitur `backend-cuda`, `backend-metal`, dan `backend-vulkan` MUST dapat diaktifkan secara terpisah.
- REQ-7-08: Build rilis MUST menggunakan `lto = "fat"`, `codegen-units = 1`, `panic = "abort"`, dan `strip = "symbols"` kecuali untuk build debugging.

---

## 8. Primitif Encoding

### 8.1 Integer

- REQ-8-01: Integer tetap MUST little-endian dengan lebar u8, u16, u32, u64, u128, i8, i16, i32, i64.
- REQ-8-02: Varint tidak bertanda MUST memakai LEB128. Panjang maksimum untuk u64 adalah 10 byte.
- REQ-8-03: Varint MUST minimal. Encoding yang memiliki byte tambahan yang tidak perlu (misalnya byte 0x80 di akhir) MUST ditolak dengan `NCI-E-FORMAT`.
- REQ-8-04: Varint bertanda MUST memakai ZigZag sebelum LEB128.
- REQ-8-05: Pembaca MUST menolak overflow pada setiap operasi aritmetika integer yang bergantung pada input dengan `NCI-E-FORMAT`.

### 8.2 Floating Point

- REQ-8-06: FP32 dan FP64 MUST mengikuti IEEE 754-2019 binary32 dan binary64.
- REQ-8-07: FP16 MUST mengikuti IEEE 754 binary16. BF16 MUST berupa 16 bit atas dari FP32 dengan pembulatan round-to-nearest-even.
- REQ-8-08: Saat canonicalization, setiap NaN MUST dinormalisasi ke pola `0x7FC00000` (FP32) atau `0x7FF8000000000000` (FP64). Nol negatif MUST dinormalisasi ke nol positif (`+0.0`) hanya pada jalur hashing canonical, bukan pada nilai komputasi.

### 8.3 Struktur Canonical TLV

Metadata dan deskriptor memakai TLV canonical:

```
TLV := tag:u16 | length:varint | value:[u8; length]
```

- REQ-8-09: Entri TLV dalam satu objek MUST diurutkan menaik berdasarkan `tag`. Tag duplikat MUST ditolak.
- REQ-8-10: Tag yang tidak dikenal oleh pembaca dengan versi minor yang sama MUST diabaikan jika bit `critical` (bit 15 dari tag) bernilai 0. Jika bit `critical` bernilai 1, pembaca MUST menolak dengan `NCI-E-SCHEMA`.
- REQ-8-11: Tag tidak boleh bernilai 0.

---

## 9. Identitas dan Hashing

### 9.1 Algoritma

- REQ-9-01: Seluruh identitas (PayloadId, RepresentationId, CellId, SchemaId, RevisionId, ManifestHash, SuperblockHash, BlockChecksum untuk payload) MUST memakai BLAKE3-256 kecuali CRC32C yang dinyatakan secara eksplisit.
- REQ-9-02: Setiap hash MUST memakai domain separator yang berbeda agar tidak terjadi tabrakan antar jenis objek.

### 9.2 Domain Separator

| Objek | Domain separator (byte ASCII, diakhiri 0x00) |
|---|---|
| Skema | `NCI/SCHEMA/v1` |
| Payload | `NCI/PAYLOAD/v1` |
| Representasi | `NCI/REP/v1` |
| Cell logis | `NCI/CELL/v1` |
| Revisi | `NCI/REV/v1` |
| Manifest | `NCI/MANIFEST/v1` |
| Superblock | `NCI/SUPER/v1` |

### 9.3 Rumus Identitas

Notasi: `||` berarti konkatenasi byte. `LP(x)` berarti `u32_le(len(x)) || x`.

```
PayloadId = BLAKE3( "NCI/PAYLOAD/v1\0" || u16(payload_type)
                    || LP(canonical_descriptor)
                    || u64(raw_len) || raw_bytes )

SchemaId  = BLAKE3( "NCI/SCHEMA/v1\0" || LP(canonical_schema_definition) )

LogicalCellId = BLAKE3( "NCI/CELL/v1\0" || u16(kind) || SchemaId
                        || u32(dep_count) || dep_1 || ... || dep_n
                        || LP(canonical_attributes)
                        || PayloadId_source )

RepresentationId = BLAKE3( "NCI/REP/v1\0" || LogicalCellId
                           || u16(rep_kind) || PayloadId_rep )
```

Aturan tambahan:

- REQ-9-03: Daftar dependensi MUST diurutkan menaik berdasarkan byte CellId sebelum hashing. Duplikasi MUST ditolak.
- REQ-9-04: `PayloadId_source` adalah payload dari representasi sumber (biasanya FP32 atau BF16 kanonik). Setiap Logical Cell MUST memiliki tepat satu representasi sumber.
- REQ-9-05: `canonical_attributes` MUST hanya berisi atribut deterministik. Atribut yang berubah seiring waktu (timestamp, skor penting, hitungan akses) MUST NOT berada di dalam atribut Cell. Atribut semacam itu disimpan di tabel metadata per revisi (Bagian 23.4).
- REQ-9-06: Setiap Cell MUST memiliki payload, meskipun payload-nya kosong (`raw_len = 0`).
- REQ-9-07: `canonical_descriptor` untuk tensor MUST berisi dtype, rank, dimensi, layout, dan skema kuantisasi. Untuk payload non-tensor, deskriptor MUST berisi tipe konten dan versi skema konten.

### 9.4 Identitas Ketika Konten Berubah

- REQ-9-08: Jika salah satu input hashing berubah (payload, dependensi, skema, atribut), CellId MUST berubah. Cell lama MUST tetap ada selama revisi yang mereferensikannya belum dihapus melalui GC.
- REQ-9-09: Kompresi, lokasi fisik, nama berkas, dan offset sumber MUST NOT memengaruhi identitas (lihat REQ-9-02 dan Bagian 18).

### 9.5 Verifikasi Identitas

- REQ-9-10: Saat setiap payload dibaca, pembaca MUST menghitung ulang PayloadId dan membandingkannya dengan indeks. Ketidakcocokan MUST menghasilkan `NCI-E-CORRUPT` dan quarantine (Bagian 27).
- REQ-9-11: Verifikasi CellId dan RepresentationId MUST dilakukan saat `ncl-check` dan saat pembukaan Cell pertama kali per sesi. Pemeriksaan ulang MUST dapat dilakukan dengan `--deep`.

---

## 10. Model Cell

### 10.1 Struktur Logis

```rust
struct Cell {
    id: LogicalCellId,              // 32 byte
    kind: CellKind,                 // u16
    schema: SchemaId,               // 32 byte
    dependencies: Vec<LogicalCellId>, // terurut
    attributes: CanonicalAttrs,     // TLV
    representations: Vec<RepresentationRef>,
    source: PayloadRef,
}
```

### 10.2 Jenis Cell (CellKind)

| Kode | Nama | Keterangan |
|---|---|---|
| 0x0001 | Weight | Bobot linear atau tensor besar |
| 0x0002 | Embedding | Tabel embedding token |
| 0x0003 | AttentionQ | Proyeksi Q |
| 0x0004 | AttentionK | Proyeksi K |
| 0x0005 | AttentionV | Proyeksi V |
| 0x0006 | AttentionO | Proyeksi output |
| 0x0007 | MlpGate | Proyeksi gate SwiGLU |
| 0x0008 | MlpUp | Proyeksi up SwiGLU |
| 0x0009 | MlpDown | Proyeksi down SwiGLU |
| 0x000A | Expert | Bobot satu expert MoE |
| 0x000B | ExpertRouter | Matriks gating MoE |
| 0x000C | Normalization | Bobot RMSNorm |
| 0x000D | LmHead | Proyeksi keluaran |
| 0x0010 | Routing | Tabel kunci routing |
| 0x0011 | Composition | Susunan Cell menjadi unit |
| 0x0012 | Execution | Definisi operator dan parameter |
| 0x0013 | Policy | Kebijakan eksekusi default |
| 0x0020 | Memory | Konten memori (umum) |
| 0x0021 | SemanticMemory | Fakta dan relasi |
| 0x0022 | EpisodicMemory | Kejadian atau contoh |
| 0x0023 | ProceduralMemory | Pola yang dipelajari |
| 0x0030 | Tokenizer | Definisi tokenizer (Bagian 21.5) |
| 0x0031 | Metadata | Metadata model |
| 0x0032 | Provenance | Asal-usul konversi dan penandatanganan |
| 0x0033 | Index | Cell indeks khusus |
| 0x00FF | Custom | Jenis tambahan; MUST memiliki skema terdaftar |

- REQ-10-01: Pembaca MUST menerima semua jenis di atas. Jenis yang tidak dikenal MUST diperlakukan sebagai `Custom` hanya jika skemanya terdaftar di manifest. Jika tidak, `NCI-E-SCHEMA`.
- REQ-10-02: Runtime MUST NOT mengasumsikan Cell hanya berisi bobot.

### 10.3 Ukuran Cell

- REQ-10-03: Cell data (Weight, Embedding, Expert, Memory) MUST berukuran antara 64 KiB dan 512 MiB. Target ukuran adalah 4 MiB.
- REQ-10-04: Cell metadata (Routing, Execution, Policy, Metadata, Provenance, Index, Tokenizer) BOLEH lebih kecil dari 64 KiB.
- REQ-10-05: Planner Cell MUST menentukan batas berdasarkan lokalitas tensor, pola akses, overhead kuantisasi, ukuran cache target, dan topologi dependensi. Pemotongan acak dilarang.
- REQ-10-06: Setiap Cell MUST memiliki batas semantik yang dapat dijelaskan oleh atributnya (misalnya `layer_index`, `projection`, `expert_index`).

### 10.4 Cell Komposit dan Tile

- REQ-10-07: Tensor yang lebih besar dari batas maksimum Cell MUST dipecah menjadi Tile. Setiap Tile adalah Cell tersendiri dengan atribut `tile_shape` dan `tile_offset`.
- REQ-10-08: Cell Composition untuk tensor MUST menyimpan daftar CellId Tile di payload-nya (TileMap, Bagian 16.2).

---

## 11. Model Kolom

### 11.1 Prinsip

- REQ-11-01: Setiap region data MUST berisi kolom. Satu kolom adalah array homogen dengan `row_count` baris.
- REQ-11-02: Baris pada tabel Cell diurutkan menaik berdasarkan byte LogicalCellId.
- REQ-11-03: Pembacaan Cell lengkap MUST merekonstruksi baris dengan join berdasarkan indeks baris, tanpa scan seluruh tabel.

### 11.2 Kolom Tabel Cell

| ID | Nama | Tipe | Encoding wajib | Keterangan |
|---|---|---|---|---|
| 0x0101 | CELL_ID | [u8;32] | RAW | LogicalCellId |
| 0x0102 | CELL_KIND | u16 | RAW atau RLE | Jenis Cell |
| 0x0103 | SCHEMA_ID | [u8;32] | RAW atau DICT | SchemaId |
| 0x0104 | SOURCE_PAYLOAD | [u8;32] | RAW | PayloadId sumber |
| 0x0105 | DEP_OFFSET | u64 | DELTA_VARINT | Awal dependensi di DEP_DATA |
| 0x0106 | DEP_DATA | [u8;32] | RAW | Dependensi terurut |
| 0x0107 | ATTR_OFFSET | u64 | DELTA_VARINT | Awal atribut di ATTR_DATA |
| 0x0108 | ATTR_DATA | byte | RAW | Atribut TLV canonical |
| 0x0109 | REP_START | u32 | DELTA_VARINT | Awal baris representasi |
| 0x010A | REP_COUNT | u16 | VARINT | Jumlah representasi |
| 0x010B | FLAGS | u8 | BITPACK | Bit flag (lihat 11.4) |

Kolom payload, representasi, dan indeks didefinisikan pada tabel berikut:

| ID | Nama | Tipe | Encoding | Keterangan |
|---|---|---|---|---|
| 0x0201 | REP_ID | [u8;32] | RAW | RepresentationId |
| 0x0202 | REP_KIND | u16 | RAW | FP32, BF16, FP16, FP8E4M3, FP8E5M2, INT8, INT4, INT3 |
| 0x0203 | REP_DTYPE | u8 | RAW | Dtype logis |
| 0x0204 | REP_QUANT_GROUP | u32 | VARINT | Ukuran grup kuantisasi (0 jika tidak ada) |
| 0x0205 | REP_PAYLOAD | [u8;32] | RAW | PayloadId representasi |
| 0x0206 | REP_SCALE_REF | [u8;32] | RAW | LogicalCellId skala (atau nol) |
| 0x0207 | REP_ZERO_REF | [u8;32] | RAW | LogicalCellId zero-point (atau nol) |
| 0x0301 | PAY_ID | [u8;32] | RAW | PayloadId |
| 0x0302 | PAY_BLOCK_START | u64 | DELTA_VARINT | Blok pertama |
| 0x0303 | PAY_BLOCK_COUNT | u32 | VARINT | Jumlah blok |
| 0x0304 | PAY_RAW_SIZE | u64 | VARINT | Ukuran tidak terkompresi |
| 0x0305 | PAY_DESC | byte | RAW | Deskriptor canonical |
| 0x0401 | HASH_KEY | [u8;32] | RAW | Kunci indeks (CellId, RepId, atau PayloadId) |
| 0x0402 | HASH_SLOT_ROW | u64 | RAW | Baris target |

- REQ-11-04: Writer MUST menulis kolom dengan encoding wajib pada tabel di atas. Writer boleh memilih encoding opsional dari Bagian 11.3 jika lebih kecil.
- REQ-11-05: Reader MUST mendukung semua encoding pada Bagian 11.3 untuk NCL versi 1.x.

### 11.3 Encoding Kolom

- **RAW:** byte tanpa transformasi.
- **RLE:** pasangan `(nilai, jumlah_run:varint)`. Nilai ditulis dalam lebar kolom.
- **DELTA_VARINT:** elemen pertama disimpan apa adanya, sisanya berupa selisih dengan elemen sebelumnya, ZigZag, lalu VARINT.
- **DELTA_RLE:** DELTA_VARINT yang hasilnya dikompres dengan RLE.
- **BITPACK:** bit dikemas little-endian dalam byte. Bit ke-0 adalah bit terendah.
- **VARINT:** LEB128 per elemen.
- **DICT:** tabel kamus `(count:varint, entries...)` diikuti indeks `u16` atau `u32` sesuai ukuran kamus.
- **SPARSE:** pasangan `(indeks_delta:varint, nilai)` untuk kolom mayoritas nol.

- REQ-11-06: Setiap encoding MUST memiliki uji round-trip dan vektor uji tetap.
- REQ-11-07: Reader MUST menolak encoding yang menghasilkan ukuran dekode melebihi `raw_size` yang dinyatakan.

### 11.4 Flag Cell

| Bit | Nama | Makna |
|---|---|---|
| 0 | QUANTIZED | Ada representasi terkuantisasi |
| 1 | COMPRESSED | Payload terkompresi (informasi, bukan identitas) |
| 2 | DEDUP_SHARED | Payload dipakai bersama |
| 3 | HAS_GRAPH_EDGES | Memiliki edge di tabel graph |
| 4 | ROUTABLE | Memiliki kunci routing |
| 5 | MEMORY_BACKED | Memiliki baris metadata memori |
| 6 | ENCRYPTED | Payload terenkripsi (Bagian 28.4) |
| 7 | QUARANTINED | Dinyatakan rusak oleh verifikasi (hanya di indeks lokal) |

---

## 12. Format Berkas NCL

### 12.1 Ikhtisar Tata Letak

Berkas `.ncl` v1 adalah urutan byte dengan tata letak berikut:

```
Offset 0        ┌────────────────────────────┐
                │ Superblock slot A (4096 B) │
Offset 4096     ├────────────────────────────┤
                │ Superblock slot B (4096 B) │
Offset 8192     ├────────────────────────────┤
                │ Region-region (append-only)│
                │  - MANIFEST                │
                │  - CELL_COLUMNS            │
                │  - GRAPH                   │
                │  - ROUTING                 │
                │  - MEMORY                  │
                │  - REPRESENTATION          │
                │  - PAYLOAD (blok)          │
                │  - INDEX                   │
                │  - REVISION                │
                │  - PROVENANCE              │
                │  - TOKENIZER               │
                │  - INTEGRITY (tanda tangan)│
                └────────────────────────────┘
```

- REQ-12-01: Berkas MUST dimulai dengan dua slot superblock berukuran masing-masing 4096 byte.
- REQ-12-02: Region baru MUST selalu ditulis di akhir berkas. Region lama MUST NOT ditimpa, kecuali oleh proses repack.
- REQ-12-03: Region MUST dimulai pada offset yang kelipatan 8.

### 12.2 Superblock (4096 byte)

| Offset | Ukuran | Field | Keterangan |
|---|---|---|---|
| 0 | 4 | magic | `4E 43 4C 01` ("NCL" + versi format 1) |
| 4 | 2 | major_version | MUST 1 |
| 6 | 2 | minor_version | MUST ≥ 0 |
| 8 | 4 | feature_flags | bit 0 terenkripsi, bit 1 bertanda tangan, bit 2 memiliki graph, bit 3 memiliki memori |
| 12 | 1 | endianness | MUST 1 |
| 13 | 1 | hash_algorithm | MUST 1 (BLAKE3-256) |
| 14 | 2 | header_size | MUST 4096 |
| 16 | 8 | file_size | Ukuran berkas saat komit ini |
| 24 | 8 | commit_generation | Nomor komit monoton |
| 32 | 8 | manifest_offset | Offset region MANIFEST |
| 40 | 8 | manifest_size | Ukuran data MANIFEST |
| 48 | 4 | region_count | Jumlah entri direktori (≤ 16) |
| 52 | 4 | reserved | MUST 0 |
| 56 | 8 | created_unix_ms | Waktu komit (informasi, bukan identitas) |
| 64 | 192 | reserved | MUST 0 |
| 256 | 896 | region_directory | 16 entri × 56 byte (lihat 12.3) |
| 1152 | 32 | manifest_hash | BLAKE3-256 dari seluruh data MANIFEST |
| 1184 | 32 | superblock_hash | BLAKE3-256 dari byte 0..1184 slot ini |
| 1216 | 2880 | reserved | MUST 0 |

- REQ-12-04: `superblock_hash` MUST dihitung dengan domain `NCI/SUPER/v1` atas byte 0 sampai 1183.
- REQ-12-05: Reader MUST memvalidasi magic sebagai operasi pertama. Kegagalan MUST menghasilkan `NCI-E-FORMAT`.
- REQ-12-06: Reader MUST memvalidasi `superblock_hash` pada kedua slot. Slot yang tidak valid diabaikan.
- REQ-12-07: Jika kedua slot valid, reader MUST memilih slot dengan `commit_generation` lebih besar. Jika keduanya bernilai sama dengan hash berbeda, reader MUST menolak dengan `NCI-E-CORRUPT`.
- REQ-12-08: Jika tidak ada slot valid, reader MUST menolak dengan `NCI-E-CORRUPT`.

### 12.3 Direktori Region

Setiap entri (56 byte):

| Offset | Ukuran | Field |
|---|---|---|
| 0 | 4 | region_type (lihat 12.4) |
| 4 | 4 | region_flags |
| 8 | 8 | offset |
| 16 | 8 | size (termasuk header region) |
| 24 | 32 | region_hash (BLAKE3-256 dari seluruh bytes region) |

- REQ-12-09: Entri direktori MUST diurutkan menaik berdasarkan `region_type` lalu `offset`.
- REQ-12-10: Setiap tipe region MUST hanya memiliki satu instance aktif pada satu superblock. Instance lama tetap ada sebagai riwayat revisi.

### 12.4 Tipe Region

| Kode | Nama |
|---|---|
| 0x01 | MANIFEST |
| 0x02 | CELL_COLUMNS |
| 0x03 | GRAPH |
| 0x04 | ROUTING |
| 0x05 | MEMORY |
| 0x06 | REPRESENTATION |
| 0x07 | PAYLOAD |
| 0x08 | INDEX |
| 0x09 | REVISION |
| 0x0A | PROVENANCE |
| 0x0B | TOKENIZER |
| 0x0C | INTEGRITY |
| 0x0D | TABLE_METADATA (per revisi, Bagian 23.4) |

### 12.5 Header Region (64 byte)

| Offset | Ukuran | Field |
|---|---|---|
| 0 | 4 | magic `NRG1` |
| 4 | 4 | region_type |
| 8 | 4 | region_version |
| 12 | 4 | flags |
| 16 | 8 | data_size |
| 24 | 8 | entry_count |
| 32 | 32 | reserved (0) |

- REQ-12-11: Pembaca MUST memvalidasi `magic`, `region_type` yang cocok dengan direktori, dan `data_size` yang sesuai dengan ukuran di direktori.

### 12.6 Blok Payload

Setiap blok (dimulai pada kelipatan 8):

| Offset | Ukuran | Field |
|---|---|---|
| 0 | 32 | payload_id |
| 32 | 4 | slice_index |
| 36 | 4 | slice_count |
| 40 | 8 | raw_size |
| 48 | 8 | stored_size |
| 56 | 1 | codec (0=none, 1=zstd) |
| 57 | 3 | reserved |
| 60 | 4 | crc32c dari bytes terkompresi |
| 64 | n | data terkompresi (stored_size byte) |

- REQ-12-12: Ukuran blok target: minimum 64 KiB, default 1 MiB, maksimum 16 MiB. Payload yang lebih besar dipecah menjadi beberapa blok dengan `slice_count` lebih dari 1.
- REQ-12-13: Slice MUST dikompresi secara independen sehingga pembacaan selektif tidak memerlukan dekompresi seluruh payload.
- REQ-12-14: `crc32c` adalah deteksi cepat, bukan mekanisme keamanan. Verifikasi autoritatif tetap menggunakan PayloadId.

---

## 13. Indeks

### 13.1 Struktur

- REQ-13-01: Region INDEX MUST berisi empat tabel hash open-addressing: `CELL_INDEX`, `REP_INDEX`, `PAYLOAD_INDEX`, `TILE_INDEX`.
- REQ-13-02: Tabel hash MUST memiliki kapasitas berupa pangkat dua, dengan faktor muat maksimum 0,5.
- REQ-13-03: Fungsi hash slot adalah 8 byte pertama kunci (little-endian) dibagi kapasitas (dengan AND karena kapasitas pangkat dua). Probing MUST linear.
- REQ-13-04: Slot kosong MUST berisi kunci nol. Kunci nol MUST NOT valid sebagai identitas.
- REQ-13-05: Setiap slot berisi `HASH_KEY` (32 byte) dan `HASH_SLOT_ROW` (8 byte).

### 13.2 Kompleksitas

- REQ-13-06: Pencarian indeks MUST O(1) rata-rata di bawah asumsi distribusi uniform dari identitas BLAKE3.
- REQ-13-07: Lookup alternatif (misalnya dari TileId ke blok) MUST diimplementasikan lewat tabel tambahan dengan kunci yang sama struktur.

### 13.3 Verifikasi Indeks

- REQ-13-08: Saat pembukaan berkas, pembaca MUST memverifikasi `region_hash` untuk INDEX secara penuh. Verifikasi penuh tidak boleh dilewati, dan spot check tidak mencukupi.
- REQ-13-09: Pembaca MUST memvalidasi bahwa setiap `HASH_SLOT_ROW` berada dalam batas tabel target.
- REQ-13-10: Pembaca MUST memvalidasi bahwa setiap entri indeks yang mengarah ke baris memiliki kunci yang sama dengan kunci di tabel target (kecuali untuk tabel TILE_INDEX yang kuncinya merujuk ke tabel CELL).

---

## 14. Manifest dan Revisi

### 14.1 Manifest

Manifest adalah region MANIFEST yang berisi TLV canonical (Bagian 8.3) dengan tag berikut:

| Tag | Tipe | Keterangan |
|---|---|---|
| 0x0001 | [u8;32] | model_id |
| 0x0002 | [u8;32] | revision_id |
| 0x0003 | u64 | cell_count |
| 0x0004 | u64 | representation_count |
| 0x0005 | u64 | payload_count |
| 0x0006 | u64 | block_count |
| 0x0007 | [u8;32] | graph_root |
| 0x0008 | [u8;32] | routing_root |
| 0x0009 | [u8;32] | memory_root |
| 0x000A | [u8;32] | index_root |
| 0x000B | u16 | ncl_major |
| 0x000C | u16 | ncl_minor |
| 0x000D | u32 | feature_flags |
| 0x000E | u64 | commit_generation |
| 0x000F | [u8;32] | parent_revision_id (opsional, bisa lebih dari satu) |
| 0x0010 | string | model_name (UTF-8, maks 256 byte) |
| 0x0011 | string | model_version_label |
| 0x0012 | [u8;32] | tokenizer_cell_id |

- REQ-14-01: `root_hash` (manifest_hash di superblock) MUST = BLAKE3-256 dari seluruh bytes data MANIFEST dengan domain `NCI/MANIFEST/v1`.
- REQ-14-02: `model_id` MUST diturunkan dari `LogicalCellId` Cell Metadata pertama pada revisi pertama. `model_id` MUST tidak berubah sepanjang DAG revisi.
- REQ-14-03: Manifest MUST memuat nilai `commit_generation` yang sama dengan superblock yang menunjuknya.

### 14.2 Revisi

Region REVISION berisi record revisi:

```
RevisionRecord := {
    revision_id:    [u8;32],     // BLAKE3("NCI/REV/v1\0" || sorted_parents || manifest_hash || LP(message))
    parents:        [[u8;32]],   // terurut, 0 sampai 4 parent
    manifest_hash:  [u8;32],
    commit_generation: u64,
    author_label:   string,      // maks 128 byte, tidak dijadikan kepercayaan
    message:        string,      // maks 4096 byte
    change_summary: TLV          // jumlah Cell ditambah, diubah, dihapus
}
```

- REQ-14-04: Setiap komit MUST menambahkan tepat satu RevisionRecord.
- REQ-14-05: Revisi MUST tidak pernah dihapus dari DAG kecuali melalui repack yang menghasilkan berkas baru dan dicatat sebagai revisi pertama berkas tersebut dengan parent kosong dan atribut `squashed_from`.
- REQ-14-06: Sebuah revisi MAY memiliki lebih dari satu parent (merge).
- REQ-14-07: Pembaca MUST memverifikasi `revision_id` setiap record. Ketidakcocokan MUST menghasilkan `NCI-E-REVISION`.

### 14.3 Konflik Merge

- REQ-14-08: Dua revisi dianggap konflik jika keduanya menambah, mengubah, atau menghapus LogicalCellId yang sama dengan hasil berbeda, atau jika keduanya mengubah entri routing atau memori yang sama.
- REQ-14-09: Operasi merge MUST gagal dengan `NCI-E-MERGE` dan menampilkan daftar konflik. Resolusi MUST dilakukan dengan membuat revisi baru yang sudah memilih satu sisi atau menggabungkan secara eksplisit.
- REQ-14-10: Merge otomatis hanya diizinkan untuk perubahan yang saling tidak bersinggungan (himpunan Cell disjoint).

---

## 15. Penulisan, Pembacaan, Komit, dan Konkurensi

### 15.1 Model Lock

- REQ-15-01: Writer MUST memperoleh lock eksklusif advisory (`flock` pada Unix, `LockFileEx` pada Windows) pada berkas sebelum menulis. Jika lock tidak tersedia dalam waktu 5 detik (default, dapat dikonfigurasi), writer MUST gagal dengan `NCI-E-LOCK`.
- REQ-15-02: Readers MUST NOT memerlukan lock eksklusif. Reader MUST membaca slot superblock dan region yang dirujuk oleh slot tersebut. Karena berkas bersifat append-only, data yang dirujuk tidak akan berubah selama reader membacanya.
- REQ-15-03: Reader MUST menyimpan `commit_generation` yang dibukanya. Reader yang membuka berkas tidak akan terpengaruh oleh komit baru hingga ia dibuka ulang.

### 15.2 Urutan Komit

Setiap komit MUST mengikuti urutan berikut:

1. Acquire lock eksklusif.
2. Baca superblock aktif dan validasi.
3. Tulis semua region baru (payload, kolom, indeks, revisi, manifest) di akhir berkas.
4. `fsync` (atau `F_FULLFSYNC` di macOS) pada berkas.
5. Susun superblock baru dengan `commit_generation + 1` dan `superblock_hash`.
6. Tulis superblock baru ke slot yang TIDAK aktif.
7. `fsync` lagi.
8. Lepas lock.

- REQ-15-04: Langkah 6 MUST menulis tepat 4096 byte pada offset slot yang tidak aktif. Jika proses gagal sebelum langkah 7 selesai, slot yang lama tetap aktif karena generasinya lebih kecil atau hash-nya tidak valid.
- REQ-15-05: Writer MUST NOT memodifikasi slot aktif pada komit apapun.
- REQ-15-06: Pada kegagalan apapun sebelum langkah 6, writer MUST menghapus data yang belum terkomit dengan memotong berkas ke ukuran sebelum komit (`ftruncate`). Pemotongan MUST diikuti `fsync`.
- REQ-15-07: Jika terjadi crash saat pemotongan, reader tetap aman karena hanya superblock yang menentukan keadaan.

### 15.3 Atomisitas

- REQ-15-08: Komit MUST bersifat all-or-nothing. Pembaca MUST melihat keadaan sebelum komit atau sesudah komit, tidak ada keadaan setengah jadi.
- REQ-15-09: Writer MUST memvalidasi seluruh data yang akan ditulis (skema, referensi, hash, batas ukuran) sebelum langkah 3. Kegagalan validasi MUST membatalkan komit tanpa menyentuh berkas.

### 15.4 Streaming

- REQ-15-10: Writer MUST menerima input secara streaming. Setidaknya satu tensor, dan satu blok payload, harus berada di memori pada satu waktu. Writer MUST NOT menahan seluruh model di memori.
- REQ-15-11: Penulisan payload MUST dilakukan per blok. Hashing MUST dilakukan secara inkremental dengan `blake3::Hasher`.
- REQ-15-12: Batas memori penulisan MUST dapat dikonfigurasi (`converter.max_resident_bytes`, default 2 GiB). Melebihi batas MUST memicu flush blok.

### 15.5 Antarmuka Reader

```rust
pub trait NclReader {
    fn open(path: &Path, opts: OpenOptions) -> Result<Self, NciError>;
    fn generation(&self) -> u64;
    fn manifest(&self) -> &Manifest;
    fn resolve_cell(&self, id: &LogicalCellId) -> Result<CellRow, NciError>;
    fn read_cell(&self, id: &LogicalCellId) -> Result<Cell, NciError>;
    fn read_representation(&self, id: &RepresentationId) -> Result<PayloadBytes, NciError>;
    fn read_payload(&self, id: &PayloadId) -> Result<PayloadBytes, NciError>;
    fn read_column(&self, col: ColumnId) -> Result<ColumnView, NciError>;
    fn read_block(&self, id: BlockId) -> Result<BlockBytes, NciError>;
    fn verify(&self, depth: VerifyDepth) -> Result<VerifyReport, NciError>;
}
```

Versi asinkron MUST tersedia melalui trait `AsyncNclReader` dengan semantik identik.

- REQ-15-13: Setiap operasi `read_*` MUST hanya membaca region dan blok yang diperlukan.
- REQ-15-14: `read_cell` MUST mengembalikan error `NCI-E-NOTFOUND` jika Cell tidak ada di revisi yang dibuka.

### 15.6 State Machine Reader

```
CLOSED --open()--> OPENED --validate_superblock()--> VALIDATED
        --load_index()--> READY --resolve/read--> ACTIVE --close()--> CLOSED
```

- REQ-15-15: Setiap transisi MUST eksplisit. Kegagalan pada transisi MUST membawa reader ke CLOSED dan melaporkan error.

### 15.7 State Machine Writer

```
CLOSED --create()--> CREATED --begin()--> WRITING --validate()--> VALIDATING
       --finalize()--> FINALIZING --commit()--> COMMITTED
(sembarang state) --abort()--> CLOSED
```

### 15.8 Interaksi Reader dan Writer

- REQ-15-16: Reader yang membuka generasi G tidak boleh mengalami perubahan bacaan ketika writer mengomit generasi G+1.
- REQ-15-17: Pembacaan dengan reader yang terbuka lama tidak boleh menyebabkan writer gagal. Writer hanya tidak boleh memotong region yang masih dirujuk oleh superblock aktif.

---

## 16. Tensor, Tile, dan Payload

### 16.1 Deskriptor Tensor

Deskriptor canonical (TLV):

| Tag | Tipe | Keterangan |
|---|---|---|
| 0x0001 | u8 | dtype (lihat 17.1) |
| 0x0002 | u8 | rank (0 sampai 8) |
| 0x0003 | [u64] | dimensions (panjang = rank) |
| 0x0004 | u8 | layout (0=row-major, 1=column-major) |
| 0x0005 | u16 | rep_kind |
| 0x0006 | u32 | quant_group_size |
| 0x0007 | u8 | quant_axis |

- REQ-16-01: Rank MUST ≤ 8. Dimensi MUST > 0 kecuali tensor kosong yang didefinisikan secara eksplisit.
- REQ-16-02: Perkalian dimensi MUST diperiksa overflow. Overflow MUST menghasilkan `NCI-E-FORMAT`.
- REQ-16-03: Layout default adalah row-major. Layout column-major hanya didukung di konverter, dan di-normalisasi ke row-major sebelum penyimpanan.

### 16.2 TileMap

Cell komposit tensor memiliki payload TileMap:

```
TileMap := {
    tile_count: u32,
    tiles: [{ cell_id: [u8;32], offset: [u64; rank], shape: [u64; rank] }],
}
```

- REQ-16-04: Tile MUST tidak tumpang tindih dan MUST menutupi seluruh tensor tepat satu kali.
- REQ-16-05: Urutan tile MUST sesuai dengan urutan row-major dari offset.

### 16.3 Ukuran Tile

- REQ-16-06: Tile default berukuran 256 × 256 elemen untuk matriks 2D, dengan batas ukuran byte 4 MiB pada representasi sumber. Planner MUST menyesuaikan tile agar memenuhi batas ini.

---

## 17. Kuantisasi dan Representasi

### 17.1 Format Wajib

| Kode REP_KIND | Nama | Bit per elemen | Keterangan |
|---|---|---|---|
| 0x0001 | FP32 | 32 | IEEE 754 binary32 |
| 0x0002 | BF16 | 16 | Bfloat16 |
| 0x0003 | FP16 | 16 | IEEE 754 binary16 |
| 0x0004 | FP8_E4M3 | 8 | OCP FP8 E4M3 (tanpa infinity, satu NaN) |
| 0x0005 | FP8_E5M2 | 8 | OCP FP8 E5M2 |
| 0x0006 | INT8 | 8 | Integer bertanda |
| 0x0007 | INT4 | 4 | Integer bertanda, dikemas |
| 0x0008 | INT3 | 3 | Integer bertanda, dikemas |

- REQ-17-01: Setiap format di atas MUST didukung oleh writer, reader, dan runtime.
- REQ-17-02: Penambahan format memerlukan amandemen (Bagian 38).

### 17.2 Kuantisasi Per Grup

- REQ-17-03: Kuantisasi INT8, INT4, dan INT3 MUST simetris atau asimetris per grup. Grup default berukuran 128 elemen sepanjang sumbu `quant_axis`.
- REQ-17-04: Skala MUST disimpan sebagai Cell terpisah (`scale_ref`) dengan representasi FP32. Zero point (`zero_ref`) MUST disimpan dengan format INT8 atau INT4.
- REQ-17-05: Rumus kuantisasi simetris: `q = clamp(round_half_to_even(x / s), qmin, qmax)` dengan `s = max(|x|) / qmax` per grup. Jika `max(|x|) = 0`, `s = 1.0`.
- REQ-17-06: Rumus dekuantisasi: `x' = (q - z) × s` dalam FP32.
- REQ-17-07: Rentang: INT8 `[-128, 127]`, INT4 `[-8, 7]`, INT3 `[-4, 3]`. Pada kuantisasi simetris, `qmin` dan `qmax` MUST disesuaikan agar simetris secara eksplisit (misalnya INT4 simetris menggunakan `[-7, 7]`).

### 17.3 Pengemasan Bit

- REQ-17-08: INT4 dikemas dua elemen per byte. Elemen indeks genap berada di nibble rendah (bit 0 sampai 3).
- REQ-17-09: INT3 dikemas delapan elemen per 3 byte dengan urutan bit little-endian. Elemen ke-i menempati bit `[3i, 3i+2]` dari aliran 24-bit per kelompok.
- REQ-17-10: Panjang payload INT3 dan INT4 MUST dibulatkan ke byte terdekat di atas. Padding MUST nol.

### 17.4 FP8

- REQ-17-11: FP8 E4M3 dan E5M2 MUST mengikuti spesifikasi OCP. Konversi dari FP32 MUST memakai round-to-nearest-even dan saturasi pada nilai maksimum yang didefinisikan format.

### 17.5 Representasi Acuan

- REQ-17-12: Setiap Logical Cell MUST memiliki representasi FP32 atau BF16 sebagai sumber (`PayloadId_source`).
- REQ-17-13: Representasi lain MUST diturunkan dari sumber menggunakan kuantisasi deterministik. Hasil kuantisasi yang sama dari sumber yang sama MUST menghasilkan PayloadId yang sama.

### 17.6 Pemilihan Representasi Runtime

- REQ-17-14: Runtime MUST memilih representasi berdasarkan: dukungan backend, budget byte, kebijakan kualitas (`min_precision` per jenis Cell), dan ketersediaan di berkas.
- REQ-17-15: Jika tidak ada representasi yang memenuhi `min_precision`, runtime MUST menolak dengan `NCI-E-REPRESENTATION`. Runtime MUST NOT mengambil representasi di bawah kebijakan secara diam-diam.

---

## 18. Kompresi

- REQ-18-01: Codec wajib: `none` (0) dan `zstd` (1). Codec zstd MUST memakai level default 3 untuk konversi, dan level dapat dikonfigurasi (1 sampai 19).
- REQ-18-02: Codec lain hanya boleh ditambahkan melalui amandemen.
- REQ-18-03: Dekompresi MUST memakai `raw_size` yang dinyatakan sebagai batas keras. Output yang melebihi batas MUST menghasilkan `NCI-E-FORMAT`.
- REQ-18-04: Dekompresi MUST melakukan streaming dengan buffer tetap. Rasio dekompresi di atas 1000:1 MUST ditolak sebagai bom dekompresi.
- REQ-18-05: Kompresi MUST NOT memengaruhi identitas (REQ-9-09).

---

## 19. Deduplikasi

- REQ-19-01: Jika dua payload memiliki PayloadId yang sama, sistem MUST menyimpannya hanya sekali dalam berkas.
- REQ-19-02: Writer MUST memeriksa indeks PAYLOAD sebelum menulis payload baru. Jika sudah ada, writer MUST hanya menambah referensi.
- REQ-19-03: Deduplikasi antar-berkas tidak dilakukan oleh NCI v1. Setiap berkas adalah unit penyimpanan independen.
- REQ-19-04: Menghapus Cell yang payload-nya dibagi MUST tidak menghapus payload fisik. Penghapusan fisik hanya terjadi melalui repack (Bagian 31, `ncl-repack`).

---

## 20. Graph

### 20.1 Struktur

Edge disimpan dalam region GRAPH dengan kolom:

| ID | Nama | Tipe | Encoding |
|---|---|---|---|
| 0x0501 | EDGE_SRC | [u8;32] | RAW (diurutkan) |
| 0x0502 | EDGE_DST | [u8;32] | RAW |
| 0x0503 | EDGE_REL | u8 | RAW |
| 0x0504 | EDGE_WEIGHT | f32 | RAW |
| 0x0505 | EDGE_FLAGS | u8 | BITPACK |
| 0x0506 | ADJ_OFFSET | u64 | DELTA_VARINT |

Jenis relasi (EDGE_REL): 0 dependency, 1 routing, 2 composition, 3 correlation, 4 causation, 5 memory_link.

- REQ-20-01: Tabel edge MUST diurutkan menaik berdasarkan `(EDGE_SRC, EDGE_REL, EDGE_DST)`.
- REQ-20-02: `ADJ_OFFSET` MUST menyimpan awal rentang edge untuk setiap sumber unik yang memiliki edge, sehingga pencarian tetangga adalah pencarian biner pada `EDGE_SRC`.
- REQ-20-03: Dependensi Cell MUST direpresentasikan sebagai edge relasi 0 dan MUST konsisten dengan kolom DEP_DATA. Jika tidak konsisten, `NCI-E-CORRUPT`.

### 20.2 Operasi

- REQ-20-04: Traversal MUST mendukung: tetangga langsung, tetangga terfilter relasi, traversal leluhur dan keturunan dengan batas kedalaman dan jumlah simpul.
- REQ-20-05: Traversal MUST menggunakan antrian dengan batas ukuran tetap. Melampaui batas MUST menghasilkan `NCI-E-BUDGET` atau hasil parsial dengan flag `truncated`.
- REQ-20-06: Graph MUST NOT dimuat seluruhnya ke RAM kecuali berkas kecil di bawah 256 MiB dan dikonfigurasi demikian.

### 20.3 Siklus Dependensi

- REQ-20-07: Graph dependensi (relasi 0) MUST acyclic. Writer MUST menolak penambahan siklus dengan `NCI-E-SCHEMA`.

---

## 21. Model Komputasi dan Eksekusi

### 21.1 Operator Wajib

Execution Cell menyimpan operator dengan opcode dan parameter TLV. Himpunan operator v1:

| Opcode | Nama | Semantik |
|---|---|---|
| 0x01 | EMBED | Lookup baris tabel embedding berdasarkan token ID |
| 0x02 | RMSNORM | `y = x / sqrt(mean(x²) + eps) × g`, eps default 1e-5 |
| 0x03 | LINEAR | `y = W x` dengan W terkuantisasi, akumulasi FP32 |
| 0x04 | ROPE | Rotary positional embedding, base default 10000 |
| 0x05 | ATTENTION | Causal self-attention dengan GQA (jumlah kepala KV ≤ kepala Q) |
| 0x06 | SWIGLU | `y = SiLU(gate(x)) ⊙ up(x)` |
| 0x07 | ADD | Penjumlahan elemen (residual) |
| 0x08 | SOFTMAX | Softmax numerik stabil (kurangi maks) |
| 0x09 | MOE_TOPK | Gating top-k atas ExpertRouter, bobot dinormalisasi softmax |
| 0x0A 0x0B | LM_HEAD | LINEAR tanpa normalisasi lanjut, menghasilkan logit |
| 0x0C | SAMPLE | Pemilihan token (greedy, temperature, top-k, top-p) |

- REQ-21-01: Setiap operator MUST memiliki implementasi referensi CPU yang deterministik (Bagian 29).
- REQ-21-02: Aktivasi (input dan output operator) MUST berada dalam FP32 pada v1. Penyimpanan KV cache MUST FP32 atau FP16 (dikonfigurasi, default FP16).
- REQ-21-03: Akumulasi dalam operator LINEAR dan ATTENTION MUST FP32.
- REQ-21-04: Operator lain di luar himpunan ini memerlukan amandemen (REQ-38).

### 21.2 Graf Model

- REQ-21-05: Model MUST didefinisikan sebagai graf Execution Cell. Setiap langkah dekoding menjalankan urutan operator yang dipilih oleh Routing (Bagian 22).
- REQ-21-06: Urutan dasar satu lapisan transformer: RMSNORM → LINEAR(Q,K,V) → ROPE → ATTENTION → LINEAR(O) → ADD → RMSNORM → (SWIGLU atau MOE_TOPK) → ADD.
- REQ-21-07: Hyperparameter model (jumlah lapisan, dimensi, jumlah kepala, dimensi kepala, dimensi FFN, jumlah expert, top-k, ukuran kosakata, konteks maksimum) MUST tersimpan di Cell Metadata dan MUST tervalidasi saat pembukaan.

### 21.3 KV Cache

- REQ-21-08: KV cache MUST bersifat ephemeral (tidak dipersistenkan sebagai Cell).
- REQ-21-09: Ukuran KV cache maksimum = `max_context × layers × 2 × kv_heads × head_dim × bytes_per_element`. Runtime MUST menolak sesi yang melampaui `max_context` dengan `NCI-E-BUDGET`.
- REQ-21-10: Pembuatan prompt (prefill) MUST memproses token dalam blok berukuran konfigurasi (default 256).

### 21.4 Pengambilan Sampel

- REQ-21-11: Sampling MUST menerima `seed` (u64). Default seed adalah 0 pada mode deterministik.
- REQ-21-12: Urutan sampling: penerapan temperature, lalu top-k, lalu top-p, lalu sampling dari distribusi dengan generator PRNG yang didefinisikan spesifikasi (ChaCha20 dengan kunci dari seed).
- REQ-21-13: Temperature 0 MUST berarti greedy argmax dengan tie-break indeks terkecil.

### 21.5 Tokenizer

- REQ-21-14: v1 MUST mendukung byte-level BPE. Tokenizer disimpan sebagai Cell Tokenizer dengan payload TLV: daftar token (UTF-8 atau byte), daftar merge berurutan, token khusus (BOS, EOS, PAD, UNK), dan aturan pre-tokenization.
- REQ-21-15: Encode dan decode MUST bersifat bit-exact terhadap definisi tokenizer. Round-trip teks untuk input UTF-8 valid MUST identik.
- REQ-21-16: Tokenizer lain (SentencePiece, WordPiece) MUST memerlukan amandemen.

---

## 22. Routing, Eksekusi Dinamis, dan Budget

### 22.1 Routing Cell

Cell Routing berisi:

- Kunci routing per Cell target (`ROUTE_TARGET`, `ROUTE_KEY` dalam FP16 atau FP32).
- Skor dasar opsional.
- Aturan kebijakan default.

### 22.2 Algoritma Seleksi

Untuk setiap langkah routing dengan vektor kueri `q` (dimensi `d`):

1. Hitung skor `s_i = dot(q, k_i) / sqrt(d) + b_i` untuk setiap kandidat `i`, dengan `b_i` skor dasar (0 jika tidak ada).
2. Urutkan kandidat menurun berdasarkan `s_i`. Jika skor sama, urutkan menaik berdasarkan LogicalCellId.
3. Ambil `K` teratas, dengan `K = min(policy.max_cells_per_step, jumlah kandidat)`.
4. Hitung bobot `w_i = softmax(s)` atas kandidat terpilih.
5. Kembalikan urutan aktivasi berdasarkan `s_i`.

- REQ-22-01: Algoritma ini MUST deterministik. Perbandingan skor MUST memakai `total_cmp` untuk menangani NaN secara konsisten.
- REQ-22-02: Kandidat dengan skor NaN MUST dibuang dan dicatat sebagai `NCI-E-ROUTING` pada log tingkat warning.

### 22.3 Eksekusi Berulang

```
Input token(s)
  → Encode ke state
  → Routing memilih Cell
  → Eksekusi Cell terpilih
  → Evaluasi:
       confidence = max softmax(logit)
       jika confidence ≥ policy.threshold → Output
       jika budget habis → Output best-effort + flag
       jika depth < max_depth → Routing ulang dengan state baru
       jika tidak → Output best-effort + flag
```

- REQ-22-03: Default `policy.threshold` adalah 0.90. Default `max_depth` adalah 4 per langkah token.
- REQ-22-04: Routing ulang MUST menggunakan state hasil langkah sebelumnya sebagai kueri.
- REQ-22-05: Setiap langkah MUST mencatat `depth`, `selected_cells`, `confidence`, dan `reason_stopped` untuk observabilitas.

### 22.4 Budget

```rust
struct ComputeBudget {
    max_depth: u32,          // default 4
    max_cells: u32,          // default 4096 per token
    max_bytes: u64,          // default 64 GiB per sesi, dibatasi oleh konfigurasi
    max_compute_flops: u64,  // default tanpa batas (0)
    max_latency_ms: u64,     // default 0 (tanpa batas)
    max_context: u32,        // default 8192
}
```

- REQ-22-06: Budget MUST diperiksa sebelum setiap eksekusi Cell. Pelampauan MUST menghentikan langkah.
- REQ-22-07: Penghentian karena budget MUST menghasilkan salah satu dari: error `NCI-E-BUDGET`, atau hasil parsial dengan `status = BudgetExhausted`, `confidence`, dan daftar token yang sudah dihasilkan. Penghentian senyap dilarang.
- REQ-22-08: Mode default API MUST mengembalikan hasil parsial dengan status (bukan error) kecuali `strict_budget = true`.

### 22.5 Batas Konservatif

- REQ-22-09: Runtime MUST memberlakukan `max_cells` dan `max_bytes` di tingkat sesi, bukan hanya per langkah.

---

## 23. Memori

### 23.1 Kategori

Memori terdiri dari:

- **Working memory:** keadaan sesi aktif (KV cache, hasil langkah sebelumnya, tail ringkas). Ephemeral.
- **Episodic memory:** pasangan (kunci, nilai) yang merekam interaksi atau contoh.
- **Semantic memory:** fakta dan relasi, disimpan sebagai entri teks atau triple.
- **Procedural memory:** pola atau langkah yang dipelajari, disimpan sebagai Cell Procedural.
- **Consolidated memory:** hasil konsolidasi entri sejenis.

### 23.2 Ukuran Working Memory

- REQ-23-01: Kapasitas working memory dalam token MUST sama dengan `max(fixed_limit, floor(K × sqrt(seq_len)))`.
- REQ-23-02: Default `fixed_limit = 2048` dan `K = 64`. Keduanya dapat dikonfigurasi melalui kebijakan.
- REQ-23-03: Working memory MUST berisi hanya: state komputasi aktif, subset riwayat terpilih (berdasarkan retrieval, Bagian 23.5), dan confidence metrics. Working memory MUST NOT menyalin seluruh konteks.

### 23.3 Isi Entri Memori

Setiap entri memori adalah Cell berjenis Episodic, Semantic, atau Procedural dengan payload:

- `text_tokens` (urutan token ID), atau `text_utf8` (opsional, untuk tampilan).
- `key_vector` (FP16, dimensi tetap per model), untuk retrieval.
- `source_session` (identitas sesi sebagai atribut, bukan waktu).

- REQ-23-04: Isi entri MUST immutable setelah komit.

### 23.4 Tabel Metadata Memori Per Revisi

Atribut dinamis entri memori (skor penting, jumlah akses, waktu terakhir diakses, status konsolidasi) disimpan dalam region TABLE_METADATA untuk setiap revisi, bukan dalam atribut Cell.

| Kolom | Tipe | Keterangan |
|---|---|---|
| MEM_CELL_ID | [u8;32] | LogicalCellId entri |
| MEM_IMPORTANCE | f16 | Skor 0 sampai 1 |
| MEM_ACCESS_COUNT | u32 | Jumlah akses |
| MEM_LAST_ACCESS_MS | u64 | Waktu UNIX milidetik |
| MEM_STATUS | u8 | 0 aktif, 1 digantikan, 2 dikonsolidasi |
| MEM_MERGED_INTO | [u8;32] | Entri hasil konsolidasi (atau nol) |

- REQ-23-05: Tabel metadata memori MUST diurutkan berdasarkan MEM_CELL_ID.
- REQ-23-06: Perubahan metadata MUST dicatat dalam revisi baru, dan nilai metadata dalam revisi lama MUST tidak berubah.

### 23.5 Retrieval

- REQ-23-07: Retrieval MUST menghitung kesamaan kosinus antara `key_vector` kueri dan entri aktif (`MEM_STATUS = 0`), mengambil top-`M` (default 8), dengan tie-break LogicalCellId menaik.
- REQ-23-08: Entri yang dipilih MUST dimasukkan ke working memory dengan batas kapasitas Bagian 23.2.

### 23.6 Penulisan dan Persistensi

- REQ-23-09: Entri baru dari sesi MUST dibuat dalam buffer sesi. Entri dikomit ke berkas hanya pada akhir sesi, pada permintaan eksplisit (`session.commit_memory`), atau saat buffer melebihi batas (default 256 entri).
- REQ-23-10: Setiap komit memori MUST menghasilkan revisi baru yang mencakup entri baru dan tabel metadata yang diperbarui.
- REQ-23-11: Memori yang belum dikomit saat sesi berakhir karena crash MUST hilang. Runtime MUST melaporkan jumlah entri hilang saat pembukaan berikutnya jika log sesi tersedia.

### 23.7 Konsolidasi

- REQ-23-12: Konsolidasi berjalan hanya atas permintaan eksplisit (`nci memory consolidate`) atau sebagai tugas latar yang dapat dinonaktifkan (default: nonaktif).
- REQ-23-13: Dua entri Episodic dengan kesamaan kosinus ≥ 0,95 pada `key_vector` dan kemiripan token ≥ 0,9 (Jaccard pada himpunan token) MAY digabung. Penggabungan membuat Cell Consolidated baru dengan `source_ids` sebagai dependensi.
- REQ-23-14: Konsolidasi MUST NOT menghapus informasi: entri asal ditandai `MEM_STATUS = 2` dan `MEM_MERGED_INTO` menunjuk ke hasil.
- REQ-23-15: Konsolidasi MUST menghasilkan revisi baru yang dapat dibatalkan dengan membuka revisi sebelumnya.

### 23.8 Penghapusan

- REQ-23-16: Penghapusan memori MUST dilakukan dengan menandai `MEM_STATUS = 1` pada revisi baru, bukan dengan menghapus data. Penghapusan fisik hanya melalui repack.
- REQ-23-17: Perintah `nci memory forget` MUST menjelaskan bahwa penghapusan fisik memerlukan repack dan menawarkan perintah repack.

### 23.9 Pembelajaran Inkremental

- REQ-23-18: NCI v1 MUST mendukung impor Cell pengganti dari berkas `.ncl` lain yang memiliki LogicalCellId yang sama atau dependensi yang kompatibel, melalui `nci revision apply`. Impor MUST membuat revisi baru.
- REQ-23-19: NCI v1 MUST NOT melakukan pelatihan gradien. Dokumen dan produk MUST tidak menyebut kemampuan ini sebagai fitur.

---

## 24. Cache dan Pergerakan Data

### 24.1 Hierarki

Hierarki yang diasumsikan: penyimpanan (berkas NCL) → RAM (cache payload) → VRAM (cache representasi di GPU) → kernel eksekusi.

### 24.2 Kunci Cache

- REQ-24-01: Kunci cache MUST berupa `(RepresentationId)` atau `(PayloadId)`. Kunci MUST NOT berupa offset berkas, indeks blok, atau tata letak perangkat keras.
- REQ-24-02: Kapasitas cache RAM MUST dapat dikonfigurasi (default 4 GiB). Kapasitas cache VRAM default 0 (nonaktif) kecuali backend GPU terdeteksi.
- REQ-24-03: Eviksi MUST menggunakan LRU dengan bobot ukuran. Tie-break MUST deterministik.

### 24.3 Prefetch

- REQ-24-04: Prefetch MUST menggunakan graf dependensi dan urutan aktivasi terakhir untuk memperkirakan Cell berikutnya dengan prioritas tinggi, sedang, atau rendah.
- REQ-24-05: Prefetch prioritas tinggi MUST dijalankan secara asinkron dan MUST tidak melebihi budget memori cache.
- REQ-24-06: Prefetch yang tidak terpakai MUST tidak menghasilkan kesalahan, tetapi MUST dihitung di metrik `nci_prefetch_wasted_bytes`.

### 24.4 Metrik Pergerakan Data

- REQ-24-07: `bytes_moved` MUST berarti jumlah byte yang dibaca dari berkas (setelah dekompresi), ditambah byte yang ditransfer ke VRAM.
- REQ-24-08: `bytes_moved_per_token` MUST dihitung sebagai `bytes_moved / token_yang_dihasilkan` pada setiap sesi.

---

## 25. Backend Komputasi

### 25.1 Trait

```rust
pub trait ComputeBackend: Send + Sync {
    fn id(&self) -> BackendId;
    fn capabilities(&self) -> BackendCaps;
    fn upload(&self, rep: &RepresentationBytes) -> Result<DeviceBuffer, NciError>;
    fn execute(&self, op: &OpDesc, inputs: &[&DeviceBuffer], out: &mut DeviceBuffer)
        -> Result<(), NciError>;
    fn synchronize(&self) -> Result<(), NciError>;
}
```

### 25.2 Backend Wajib

- REQ-25-01: Backend `cpu` MUST tersedia pada semua target. Backend ini adalah referensi kebenaran.
- REQ-25-02: Backend `cpu` MUST menyediakan dua jalur: `portable` (deterministik, tanpa dispatch SIMD yang mengubah urutan reduksi) dan `optimized` (dispatch SIMD runtime).
- REQ-25-03: Dalam mode deterministik, backend MUST memakai jalur `portable`.
- REQ-25-04: Backend `cpu` `optimized` MUST menghasilkan hasil yang berada dalam toleransi tertentu terhadap `portable` (default batas kesalahan relatif 1e-5 per elemen) dan MUST diuji dengan batas tersebut.

### 25.3 Backend Opsional

- REQ-25-05: Backend `cuda`, `metal`, dan `vulkan` MUST dimuat secara dinamis (Bagian 3, REQ-3-05).
- REQ-25-06: Backend GPU MUST mendeklarasikan `deterministic = false` kecuali telah diuji untuk determinisme pada kombinasi perangkat dan driver tertentu.
- REQ-25-07: Backend GPU MUST mengimplementasikan operator wajib (Bagian 21.1) atau melaporkan operator yang tidak didukung. Operator yang tidak didukung MUST dieksekusi di CPU dengan penyalinan data eksplisit dan tercatat di metrik.
- REQ-25-08: Pemilihan backend MUST dapat dioverride melalui konfigurasi (`backend = "cpu"`).

### 25.4 Kesalahan Hardware

- REQ-25-09: Jika backend gagal saat inisialisasi, runtime MUST mencatat `NCI-E-HARDWARE` dan melanjutkan dengan backend CPU.
- REQ-25-10: Jika backend gagal saat eksekusi, runtime MUST menghentikan sesi dengan `NCI-E-HARDWARE` dan MUST NOT mengulang eksekusi secara diam-diam pada backend lain kecuali kebijakan fallback diaktifkan.

---

## 26. Konverter

### 26.1 Pipeline

Setiap konverter MUST menjalankan tahap berikut secara streaming:

1. **Reader** format eksternal (tensor stream).
2. **Normalizer** (dtype, layout ke row-major, nama).
3. **Semantic mapper** (memetakan nama tensor ke CellKind dan atribut lapisan).
4. **Cell planner** (menentukan batas Cell dan tile).
5. **Quantizer** (opsional, sesuai profil).
6. **Compressor** (zstd).
7. **Hasher** (PayloadId, RepresentationId, LogicalCellId).
8. **Writer** (komit ke NCL).

- REQ-26-01: Setiap tahap MUST memproses data secara bertahap. Tidak ada tahap yang memuat seluruh model.
- REQ-26-02: Konverter MUST menghasilkan laporan konversi dalam format JSON yang berisi jumlah tensor, tensor yang dilewati, peringatan, dan hash berkas input.
- REQ-26-03: Tensor yang tidak dipetakan ke CellKind MUST disimpan sebagai Cell Custom dengan skema `unmapped_tensor` dan MUST dicatat sebagai peringatan. Konverter MUST NOT membuang tensor secara diam-diam.
- REQ-26-04: Konverter MUST menolak model yang tidak memiliki hyperparameter lengkap (REQ-21-07) kecuali dijalankan dengan `--allow-partial` dan hasilnya ditandai `PARTIAL` di metadata.

### 26.2 Format yang Wajib Didukung

**PyTorch (`.pt`, `.pth`, `.bin` zip):**
- REQ-26-05: Parser MUST membaca arsip zip dan entri `data.pkl` secara manual. Unpickler MUST berupa implementasi Rust dengan allowlist: opcode dasar (`PROTO`, `FRAME`, `MEMOIZE`, `BINGET`, `BINPUT`, `TUPLE`, `DICT`, `LIST`, dan setara), serta global `collections.OrderedDict`, `torch._utils._rebuild_tensor_v2`, `torch.FloatStorage`, `torch.HalfStorage`, `torch.BFloat16Storage`, `torch.LongStorage`, dan `torch.IntStorage`.
- REQ-26-06: Opcode `REDUCE`, `BUILD`, `INST`, `OBJ`, `NEWOBJ`, dan `STACK_GLOBAL` dengan global di luar allowlist MUST ditolak dengan `NCI-E-UNSAFE`.
- REQ-26-07: Unpickler MUST memiliki batas kedalaman stack, batas jumlah objek, dan batas memo.
- REQ-26-08: Data storage tensor MUST dibaca dari entri `data/<key>` dalam zip dengan validasi ukuran terhadap deskriptor. Zip bomb MUST terdeteksi melalui rasio dan batas ukuran total.
- REQ-26-09: Format pickle lama (non-zip) MUST ditolak dengan `NCI-E-IMPORT` dan pesan yang menyarankan konversi ulang.

**Safetensors:**
- REQ-26-10: Parser MUST memvalidasi header (panjang header ≤ 100 MB, JSON valid, offset tidak tumpang tindih, dan rentang dalam batas file).
- REQ-26-11: Dtype yang didukung: F32, F16, BF16, F8_E4M3, F8_E5M2, I8, dan U8 (untuk INT8 terkuantisasi bila ada skala terkait).

**GGUF:**
- REQ-26-12: Parser MUST mendukung GGUF versi 3. Versi lain MUST ditolak dengan `NCI-E-IMPORT`.
- REQ-26-13: Tipe kuantisasi GGUF yang didukung: F32, F16, BF16, Q8_0, Q4_0, Q4_K, Q5_K, Q6_K. Tipe lain MUST ditolak dengan `NCI-E-REPRESENTATION` kecuali didekode ke FP32 secara eksplisit melalui opsi `--dequantize`.
- REQ-26-14: Tokenizer GGUF MUST diimpor ke Cell Tokenizer jika tipe tokenizer adalah BPE byte-level. Tipe lain MUST ditolak sesuai REQ-21-16.

### 26.3 Keamanan Konverter

- REQ-26-15: Konverter MUST NEVER mengeksekusi kode dari berkas input.
- REQ-26-16: Setiap dimensi, offset, dan ukuran MUST diperiksa overflow dan batas sebelum alokasi.
- REQ-26-17: Alokasi per tensor MUST dibatasi oleh `converter.max_tensor_bytes` (default 8 GiB).
- REQ-26-18: Jalur keluaran MUST dinormalisasi. Konverter MUST menolak jalur yang keluar dari direktori keluaran yang ditentukan.
- REQ-26-19: Berkas input MUST dibuka dalam mode baca dan MUST tidak pernah ditulis.

### 26.4 Isolasi

- REQ-26-20: Modul konverter MUST berada di crate terpisah (Bagian 6.2). Runtime MUST tidak memanggil fungsi konverter (REQ-6-04, REQ-6-10).

---

## 27. Integritas dan Korupsi

### 27.1 Prinsip

- REQ-27-01: Payload MUST diverifikasi sebelum dikembalikan ke runtime. Payload yang gagal verifikasi MUST NOT mencapai eksekusi (NCL-INV-009).
- REQ-27-02: Korupsi pada satu blok MUST NOT membuat seluruh berkas tidak dapat dibaca. Cell yang bergantung pada blok rusak MUST ditandai tidak tersedia.

### 27.2 Pemeriksaan Wajib

Saat pembukaan (`open`):
1. Magic superblock.
2. `superblock_hash` kedua slot.
3. `manifest_hash`.
4. `region_hash` untuk MANIFEST dan INDEX.
5. Validasi batas pointer seluruh region di direktori.

Saat `verify --deep` atau `ncl-check`:
6. `region_hash` seluruh region.
7. Checksum kolom (BLAKE3 atas data mentah kolom, lihat Bagian 11.1).
8. CRC32C setiap blok.
9. PayloadId setiap payload.
10. CellId dan RepresentationId setiap Cell.
11. `revision_id` setiap revisi.

- REQ-27-03: Saat pembukaan, pemeriksaan 1 sampai 5 MUST selesai sebelum berkas dinyatakan siap.
- REQ-27-04: Pemeriksaan lazy pada blok MUST dilakukan saat blok pertama kali dibaca.

### 27.3 Quarantine

- REQ-27-05: Saat ketidakcocokan terdeteksi, runtime MUST: (a) mencatat `NCI-E-CORRUPT` dengan identitas objek, (b) menandai objek sebagai `QUARANTINED` pada cache sesi, (c) menolak eksekusi yang bergantung pada objek tersebut, (d) menyarankan revisi sebelumnya yang sehat.
- REQ-27-06: Runtime MUST NOT melakukan perbaikan otomatis pada data yang rusak.
- REQ-27-07: Berkas yang rusak MUST tidak dimodifikasi oleh perintah apapun kecuali `repack` ke berkas baru. `repack` MUST menolak menyalin objek rusak kecuali `--drop-corrupt` dan mencatatnya di laporan.

### 27.4 Rollback

- REQ-27-08: Jika superblock aktif rusak tetapi slot lain valid, pembukaan MUST berhasil dan MUST mencatat peringatan `NCI-E-CORRUPT` pada slot yang rusak. Perintah `ncl-check` MUST menampilkan kondisi ini.
- REQ-27-09: Perintah `nci revision checkout <id>` MUST membuka revisi lama tanpa mengubah berkas.

---

## 28. Keamanan

### 28.1 Tanda Tangan Digital

- REQ-28-01: Algoritma tanda tangan adalah Ed25519 (RFC 8032) atas `manifest_hash` (32 byte).
- REQ-28-02: Region INTEGRITY MUST berisi, untuk setiap tanda tangan: `key_id` (BLAKE3 dari kunci publik, 32 byte), `signature` (64 byte), `signed_generation` (u64), dan `timestamp_ms` (informasi).
- REQ-28-03: Writer MUST mendukung penandatanganan dengan kunci privat yang dimuat dari berkas PEM atau dari stdin, tanpa menulis kunci ke log.
- REQ-28-04: Satu berkas MAY memiliki beberapa tanda tangan dari kunci berbeda.

### 28.2 Trust Store

- REQ-28-05: Trust store adalah berkas TOML yang berisi daftar `key_id`, kunci publik, dan label. Lokasi default: `~/.config/nci/trust.toml` atau sesuai konfigurasi.
- REQ-28-06: `nci check --require-signature` MUST gagal jika tidak ada tanda tangan yang valid dari kunci dalam trust store.
- REQ-28-07: `serve` MUST menolak berkas tanpa tanda tangan tepercaya, kecuali diizinkan dengan `--allow-unsigned`. Penggunaan opsi ini MUST dicatat sebagai peringatan keamanan pada setiap start.
- REQ-28-08: Mengubah trust store MUST tidak terjadi secara otomatis. Penambahan kunci hanya melalui `nci trust add` dengan konfirmasi interaktif atau flag `--yes`.

### 28.3 Verifikasi Kriptografi

- REQ-28-09: Verifikasi Ed25519 MUST memakai verifikasi "strict" (menolak kunci publik lemah dan signature non-canonical).
- REQ-28-10: Perbandingan hash dan tanda tangan MUST dilakukan secara konstan-waktu bila relevan.

### 28.4 Enkripsi Opsional

- REQ-28-11: Berkas MAY dienkripsi. Flag `ENCRYPTED` di superblock MUST ditetapkan.
- REQ-28-12: Kunci enkripsi diturunkan dari frasa sandi dengan Argon2id: memori 64 MiB, iterasi 3, paralelisme 1, salt acak 16 byte per berkas (disimpan di PROVENANCE). Parameter MUST dapat dikonfigurasi dan tersimpan di berkas.
- REQ-28-13: Setiap payload block terenkripsi dengan XChaCha20-Poly1305. Nonce acak 24 byte disimpan di header blok. Associated data MUST mencakup `payload_id`, `slice_index`, dan `block_id`.
- REQ-28-14: Enkripsi MUST melindungi payload, tabel memori, dan tokenizer. Kolom indeks MAY tetap plaintext, tetapi dokumentasi MUST menyatakan bahwa CellId dan PayloadId plaintext membocorkan kehadiran konten tertentu bagi pihak yang dapat menebak konten tersebut.
- REQ-28-15: Kehilangan frasa sandi MUST berarti data tidak dapat dipulihkan. Tidak ada kunci pemulihan bawaan.

### 28.5 Keamanan Layanan API

- REQ-28-16: Server MUST mengikat ke `127.0.0.1` secara default.
- REQ-28-17: Jika alamat bind bukan loopback, server MUST mewajibkan token bearer. Token dibuat dengan `nci serve token` dan disimpan sebagai hash BLAKE3 di konfigurasi.
- REQ-28-18: Server MUST menerapkan batas ukuran permintaan (default 1 MiB), batas waktu baca, dan batas jumlah koneksi bersamaan (default 64).
- REQ-28-19: Server MUST menerapkan TLS jika bind non-loopback, menggunakan sertifikat yang diberikan pengguna. Tanpa TLS, server MUST menolak bind non-loopback kecuali `--insecure-plain-http` dipakai.
- REQ-28-20: Header dan body log MUST tidak memuat token atau isi prompt secara default. Pencatatan prompt MUST diaktifkan secara eksplisit.

### 28.6 Pembatasan Eksekusi

- REQ-28-21: Runtime MUST tidak melakukan panggilan sistem berisiko tinggi di luar yang diperlukan (akses berkas hanya ke jalur yang diberikan, tanpa spawn proses).
- REQ-28-22: Pada Linux, runtime SEBAIKNYA menerapkan seccomp-bpf profil ketat setelah inisialisasi.
- REQ-28-23: Runtime MUST tidak memuat kode dinamis dari berkas `.ncl`. Berkas `.ncl` adalah data, bukan kode.

### 28.7 Pengaturan Data Pribadi

- REQ-28-24: Memori MUST dapat dihapus per entri atau seluruhnya (`nci memory forget --all`). Penghapusan seluruhnya MUST menghasilkan revisi baru yang berisi status hapus dan MUST diikuti repack bila pengguna meminta penghapusan fisik.
- REQ-28-25: Log dan metrik MUST tidak mencatat isi prompt, isi memori, atau nilai token secara default.

---

## 29. Determinisme

### 29.1 Ruang Lingkup Jaminan

- REQ-29-01: Dengan input yang sama, revisi yang sama, kebijakan eksekusi yang sama, dan build binary yang sama pada CPU dengan set instruksi yang sama, mode deterministik MUST menghasilkan output token yang identik bit-per-bit.
- REQ-29-02: Mode deterministik MUST mengaktifkan: jalur `portable`, urutan reduksi tetap, tidak ada paralelisme yang mengubah urutan penjumlahan, dan sampling dengan seed eksplisit.
- REQ-29-03: Lintas arsitektur CPU atau lintas build, determinisme MUST tidak dijamin. Dokumentasi produk MUST menyatakan ini.

### 29.2 Kebijakan Eksekusi

```rust
struct ExecutionPolicy {
    deterministic: bool,         // default false
    seed: u64,                   // default 0
    quantization_rounding: RoundingMode,  // default RNE
    reduction_order: ReductionOrder,      // default Sequential
    threads: u32,                // default auto; dalam deterministik = 1 untuk reduksi
    threshold: f32,              // default 0.90
    max_depth: u32,              // default 4
}
```

- REQ-29-04: Setiap kebijakan yang memengaruhi komputasi MUST tercatat dalam hasil eksekusi.
- REQ-29-05: Jika `deterministic = true` dan backend tidak mendukung determinisme, runtime MUST gagal dengan `NCI-E-HARDWARE`, bukan menurunkan jaminan secara diam-diam.

### 29.3 Pengujian Determinisme

- REQ-29-06: Setiap rilis MUST menyertakan vektor uji determinisme: prompt, revisi, kebijakan, dan token keluaran yang diharapkan. Vektor ini dijalankan oleh `nci selftest`.

---

## 30. Model Kesalahan

### 30.1 Kode Kesalahan

| Kode | Kategori | Makna | Pemulihan |
|---|---|---|---|
| NCI-E-FORMAT | Format | Format berkas atau encoding tidak valid | Tolak berkas |
| NCI-E-SCHEMA | Format | Skema atau versi tidak kompatibel | Tingkatkan pembaca atau tolak |
| NCI-E-MANIFEST | Format | Manifest tidak valid atau hash tidak cocok | Gunakan slot superblock lain |
| NCI-E-CORRUPT | Integritas | Data rusak terdeteksi | Quarantine, gunakan revisi sebelumnya |
| NCI-E-NOTFOUND | Penyimpanan | Cell, payload, atau blok tidak ada | Periksa revisi |
| NCI-E-STORE | Penyimpanan | Operasi I/O gagal | Ulangi atau laporkan |
| NCI-E-LOCK | Konkurensi | Lock tidak tersedia dalam batas waktu | Ulangi nanti |
| NCI-E-REVISION | Versi | Rantai revisi rusak | Gunakan revisi lain |
| NCI-E-MERGE | Revisi | Konflik merge | Resolusi manual |
| NCI-E-BUDGET | Runtime | Budget habis | Hasil parsial atau error sesuai mode |
| NCI-E-ROUTING | Runtime | Kegagalan routing | Fallback atau error |
| NCI-E-REPRESENTATION | Format | Representasi tidak didukung atau di bawah kebijakan | Pilih representasi lain |
| NCI-E-HARDWARE | Hardware | Backend gagal atau tidak tersedia | Gunakan CPU jika diizinkan |
| NCI-E-IMPORT | Konversi | Konversi format eksternal gagal | Periksa input |
| NCI-E-UNSAFE | Keamanan | Operasi berisiko terdeteksi | Tolak operasi |
| NCI-E-SIGNATURE | Keamanan | Tanda tangan tidak valid | Tolak berkas |
| NCI-E-TRUST | Keamanan | Kunci tidak ada di trust store | Tambah kunci secara sadar atau tolak |
| NCI-E-CONFIG | Konfigurasi | Konfigurasi tidak valid | Perbaiki konfigurasi |
| NCI-E-USAGE | Pemakaian | Argumen CLI atau API salah | Tampilkan bantuan |
| NCI-E-LIMIT | Sumber daya | Batas sumber daya terlampaui | Kurangi input atau naikkan batas |
| NCI-E-RECOVERY | Pemulihan | Pemulihan gagal | Intervensi manual |
| NCI-E-INTERNAL | Internal | Asersi internal gagal | Laporkan bug |

- REQ-30-01: Setiap error MUST memiliki salah satu kode di atas. Kode baru memerlukan amandemen.
- REQ-30-02: Error MUST membawa konteks: nama operasi, identitas objek jika ada, dan offset berkas jika relevan.
- REQ-30-03: Pesan error MUST tidak memuat isi prompt, token, atau kunci.
- REQ-30-04: Kegagalan senyap dilarang (NCI-P13).

### 30.2 Pemetaan HTTP

| Kode | HTTP |
|---|---|
| NCI-E-USAGE, NCI-E-CONFIG | 400 |
| NCI-E-SIGNATURE, NCI-E-TRUST | 403 |
| NCI-E-NOTFOUND | 404 |
| NCI-E-LOCK | 409 |
| NCI-E-BUDGET, NCI-E-LIMIT | 429 atau 413 sesuai kasus |
| NCI-E-CORRUPT, NCI-E-INTERNAL, NCI-E-RECOVERY | 500 |
| NCI-E-HARDWARE, NCI-E-STORE | 503 |
| lainnya | 422 |

### 30.3 Kode Keluar CLI

| Kode | Arti |
|---|---|
| 0 | Sukses |
| 1 | Error umum |
| 2 | Penggunaan salah |
| 3 | Berkas tidak ditemukan |
| 4 | Verifikasi gagal (tanda tangan, integritas) |
| 64 | Berkas rusak |
| 65 | Versi format tidak didukung |
| 66 | Konfigurasi salah |
| 67 | Hardware tidak tersedia |
| 255 | Error internal |

---

## 31. Antarmuka Baris Perintah

Binary: `nci`. Semua perintah MUST mendukung `--json` untuk keluaran mesin, `--quiet`, dan `--verbose`.

### 31.1 Perintah Wajib

| Perintah | Fungsi |
|---|---|
| `nci convert <input> -o <output.ncl> [--profile P] [--quant Q] [--allow-partial]` | Konversi PyTorch, Safetensors, atau GGUF ke NCL |
| `nci inspect <file.ncl> [--cells] [--revisions]` | Tampilkan struktur, metadata, dan revisi |
| `nci check <file.ncl> [--deep] [--require-signature]` | Verifikasi integritas dan tanda tangan |
| `nci dump <file.ncl> [--cell ID] [--payload ID] [--format raw\|hex\|json]` | Tampilkan isi Cell atau payload |
| `nci stats <file.ncl>` | Ukuran, jumlah Cell, rasio kompresi, dedup |
| `nci repack <file.ncl> -o <out.ncl> [--drop-corrupt] [--recompress LEVEL]` | Kompaksi dan penghapusan fisik |
| `nci benchmark <file.ncl> [--cells N] [--seed S]` | Ukur pembacaan selektif |
| `nci run <file.ncl> [--prompt TEXT\|--prompt-file F] [opsi kebijakan]` | Inferensi satu kali, keluaran ke stdout |
| `nci chat <file.ncl>` | Sesi interaktif dengan memori |
| `nci serve <file.ncl> [--bind ADDR] [--token-file F] [--allow-unsigned]` | Layanan HTTP |
| `nci memory list\|consolidate\|forget` | Manajemen memori |
| `nci revision list\|checkout\|apply\|diff` | Manajemen revisi |
| `nci sign <file.ncl> --key K` | Tambahkan tanda tangan |
| `nci trust add\|remove\|list` | Kelola trust store |
| `nci keygen --out PATH` | Buat pasangan kunci Ed25519 |
| `nci encrypt\|decrypt <file.ncl>` | Enkripsi atau dekripsi berkas |
| `nci selftest` | Jalankan uji konformansi dan determinisme bawaan |
| `nci version [--json]` | Versi, commit, target, fitur |
| `nci completions <shell>` | Berkas penyelesaian perintah |
| `nci config show\|init\|validate` | Kelola konfigurasi |

- REQ-31-01: Setiap perintah MUST mengembalikan kode keluar sesuai Bagian 30.3.
- REQ-31-02: `nci run` MUST membaca prompt dari stdin jika tidak ada flag prompt.
- REQ-31-03: `nci repack` MUST menghasilkan berkas baru dan MUST tidak memodifikasi berkas input.
- REQ-31-04: `nci revision apply` MUST menampilkan ringkasan perubahan dan meminta konfirmasi kecuali `--yes`.
- REQ-31-05: `nci memory forget` dan `nci encrypt` yang menimpa berkas MUST meminta konfirmasi kecuali `--yes`.
- REQ-31-06: Bantuan (`--help`) MUST tersedia untuk setiap perintah dan subperintah, dihasilkan dari definisi CLI yang sama dengan parser.

---

## 32. API Layanan

### 32.1 Protokol

- REQ-32-01: API MUST berbasis HTTP/1.1 dengan JSON. HTTP/2 SEBAIKNYA didukung.
- REQ-32-02: Respons streaming MUST memakai Server-Sent Events (`text/event-stream`).
- REQ-32-03: Versi API MUST berada di prefiks path `/v1/`.

### 32.2 Endpoint

| Metode | Path | Fungsi |
|---|---|---|
| GET | `/v1/health` | Status (tanpa data sensitif) |
| GET | `/v1/manifest` | Ringkasan manifest dan revisi aktif |
| GET | `/v1/revisions` | Daftar revisi |
| POST | `/v1/generate` | Pembangkitan teks (stream atau non-stream) |
| POST | `/v1/route` | Hasil routing untuk kueri (diagnostik) |
| POST | `/v1/sessions` | Buat sesi |
| DELETE | `/v1/sessions/{id}` | Akhiri sesi |
| POST | `/v1/sessions/{id}/memory/commit` | Komit memori sesi |
| GET | `/v1/memory` | Daftar memori aktif |
| DELETE | `/v1/memory/{cell_id}` | Tandai memori sebagai dihapus |
| GET | `/v1/metrics` | Metrik format Prometheus |
| GET | `/v1/cells/{cell_id}` | Metadata Cell (bukan payload) |

- REQ-32-04: `POST /v1/generate` MUST menerima: `prompt` (string) atau `token_ids`, `session_id` (opsional), `policy` (opsional, subset `ExecutionPolicy`), `max_new_tokens` (default 256, maksimum konfigurasi), dan `stream` (boolean).
- REQ-32-05: Respons non-stream MUST berisi: `text`, `token_ids`, `status` (`Complete`, `BudgetExhausted`, `Cancelled`), `confidence`, `revision_id`, dan `metrics` (ringkas).
- REQ-32-06: Respons stream MUST mengirim event `token`, `progress`, `done`, dan `error`.
- REQ-32-07: Klien yang memutus koneksi MUST menyebabkan pembatalan generasi pada langkah token berikutnya.
- REQ-32-08: Setiap permintaan MUST memiliki `request_id` (UUIDv7) yang dikembalikan dalam header `X-Request-Id`.

### 32.3 Batas

- REQ-32-09: Maksimum ukuran prompt default 32 KiB karakter atau 8192 token, sesuai konfigurasi.
- REQ-32-10: Permintaan melebihi batas MUST ditolak dengan `NCI-E-LIMIT`.

### 32.4 Spesifikasi OpenAPI

- REQ-32-11: Spesifikasi OpenAPI 3.1 MUST dihasilkan dari kode dan disertakan dalam binary, dapat diambil dengan `nci api spec`.

---

## 33. Konfigurasi

### 33.1 Sumber dan Urutan Prioritas

Urutan prioritas (tertinggi ke terendah):

1. Argumen CLI.
2. Variabel lingkungan dengan awalan `NCI_` (misalnya `NCI_SERVE_BIND`).
3. Berkas konfigurasi TOML (`--config` atau `~/.config/nci/config.toml`).
4. Default bawaan.

- REQ-33-01: Setiap nilai konfigurasi MUST memiliki nilai default dan MUST dapat divalidasi tanpa menjalankan model.
- REQ-33-02: Nilai konfigurasi yang tidak dikenal MUST menghasilkan `NCI-E-CONFIG`, bukan diabaikan.
- REQ-33-03: `nci config show` MUST menampilkan sumber setiap nilai efektif. Rahasia (token, kunci) MUST disamarkan.

### 33.2 Bagian Konfigurasi

| Bagian | Kunci utama | Default |
|---|---|---|
| `[runtime]` | `backend`, `threads`, `max_context` | `auto`, auto, 8192 |
| `[cache]` | `ram_bytes`, `vram_bytes` | 4 GiB, 0 |
| `[policy]` | `threshold`, `max_depth`, `max_cells_per_step` | 0.90, 4, 4096 |
| `[memory]` | `fixed_limit`, `K`, `top_m`, `buffer_entries` | 2048, 64, 8, 256 |
| `[security]` | `require_signature`, `trust_store` | true, `~/.config/nci/trust.toml` |
| `[serve]` | `bind`, `max_request_bytes`, `max_connections` | 127.0.0.1:8730, 1 MiB, 64 |
| `[converter]` | `max_tensor_bytes`, `max_resident_bytes`, `zstd_level` | 8 GiB, 2 GiB, 3 |
| `[log]` | `level`, `format`, `prompt_logging` | info, text, false |

- REQ-33-04: Port default layanan adalah 8730.

---

## 34. Observabilitas

### 34.1 Metrik

Metrik MUST dikumpulkan dan diekspos di `/v1/metrics` dengan format Prometheus.

| Nama | Tipe | Satuan | Prioritas |
|---|---|---|---|
| nci_cells_total | gauge | count | tinggi |
| nci_cells_active | gauge | count | tinggi |
| nci_cells_loaded | gauge | count | tinggi |
| nci_bytes_read_total | counter | bytes | tinggi |
| nci_bytes_moved_total | counter | bytes | kritis |
| nci_bytes_moved_per_token | histogram | bytes | kritis |
| nci_cache_hit_ratio | histogram | ratio | tinggi |
| nci_gpu_residency_ratio | gauge | ratio | sedang |
| nci_active_parameter_ratio | gauge | ratio | tinggi |
| nci_execution_depth | histogram | count | sedang |
| nci_route_selections_total | counter | count | sedang |
| nci_memory_bytes | gauge | bytes | sedang |
| nci_revision_count | gauge | count | rendah |
| nci_corrupt_objects_total | counter | count | kritis |
| nci_prefetch_wasted_bytes_total | counter | bytes | sedang |
| nci_tokens_generated_total | counter | count | tinggi |
| nci_requests_total | counter | count | tinggi |
| nci_request_latency_seconds | histogram | seconds | tinggi |
| nci_errors_total | counter | count (per kode) | tinggi |

- REQ-34-01: `nci_active_parameter_ratio` MUST = (byte Cell yang dieksekusi dalam langkah) ÷ (total byte bobot model), dihitung per token.
- REQ-34-02: Label metrik MUST terbatas. Label yang memuat identitas Cell, prompt, atau ID sesi MUST tidak digunakan sebagai label metrik.

### 34.2 Log

- REQ-34-03: Log MUST berformat terstruktur (JSON atau teks dengan key=value). Setiap entri MUST memiliki `ts`, `level`, `component`, `request_id` (jika ada), dan `event`.
- REQ-34-04: Log MUST tidak memuat prompt atau keluaran kecuali `prompt_logging = true`.

### 34.3 Jejak

- REQ-34-05: Eksekusi dapat menghasilkan jejak routing (`--trace FILE`) berformat JSON Lines, berisi per langkah: `depth`, `selected`, `scores_top`, `confidence`, `reason`. Jejak tidak memuat isi memori kecuali diminta.

---

## 35. Performa dan Baseline

### 35.1 Prinsip

- REQ-35-01: Semua klaim performa MUST diukur pada rig referensi yang didefinisikan di Bagian 35.4.
- REQ-35-02: Hasil benchmark MUST diarsipkan dengan commit, konfigurasi, dan spesifikasi perangkat keras.
- REQ-35-03: Angka target di bawah ini adalah **ambang penerimaan**. Ambang yang belum terukur pada rig referensi MUST diisi sebelum rilis. Rilis dengan ambang kosong MUST tidak diizinkan.

### 35.2 Kriteria Fungsional (Wajib, Tidak Bergantung Hardware)

- REQ-35-04: Pembacaan satu Cell dari berkas berukuran 1 TB MUST tidak memuat berkas penuh ke RAM. Diuji dengan batas RAM proses 512 MiB.
- REQ-35-05: Inferensi dalam mode deterministik MUST menghasilkan token yang sama dengan vektor uji (REQ-29-06).
- REQ-35-06: Komit yang terputus di tengah (diuji dengan injeksi kegagalan) MUST menyisakan berkas yang dapat dibuka dengan generasi sebelumnya.
- REQ-35-07: Setiap bit yang dibalik pada payload yang dipilih secara acak MUST terdeteksi oleh verifikasi (uji fuzz terhadap integritas).
- REQ-35-08: Konversi model PyTorch 7B MUST selesai tanpa melebihi `max_resident_bytes` + 1 GiB RSS.

### 35.3 Kriteria Performa (Terukur, Ambang Ditetapkan Pada Rig)

| Metrik | Kondisi uji | Ambang |
|---|---|---|
| Buka berkas (superblock + index) | Berkas 1 TB, SSD NVMe rig referensi | ≤ 2 detik |
| Baca satu Cell 4 MiB, cache dingin | NVMe rig referensi | ≤ 10 ms p99 |
| Baca satu Cell 4 MiB, cache hangat | RAM | ≤ 100 µs p99 |
| Konversi safetensors 7B ke NCL INT8 | CPU rig referensi | ≤ 2 jam |
| Inferensi 7B INT4 model dense, CPU | CPU rig referensi | tokens/s ≥ ambang rig |
| Inferensi model 1 TB dengan RAM 16 GiB, MoE | Rig 16 GiB RAM, NVMe | tokens/s ≥ ambang rig (tercatat) |
| Ukuran binary CPU-only | Target Linux musl | ≤ 60 MiB |

- REQ-35-09: Ambang tokens/s MUST ditetapkan per model acuan dan per rig pada tiket rilis, dengan metode pengukuran yang tercatat.
- REQ-35-10: Klaim bahwa model 1 TB berjalan di RAM 16 GiB hanya boleh dinyatakan jika kriteria fungsional REQ-35-04 dan ambang tokens/s pada rig 16 GiB terpenuhi.

### 35.4 Rig Referensi

Rig referensi MUST didokumentasikan dalam `ci/rig/README` dengan spesifikasi CPU (model dan jumlah core), RAM, NVMe (model dan kecepatan baca), kernel, dan versi toolchain. Minimal satu rig dengan CPU x86_64, satu rig aarch64, dan satu rig 16 GiB RAM untuk uji 1 TB.

### 35.5 Batas Non-Jaminan

- REQ-35-11: Dokumentasi produk MUST menyatakan bahwa latensi, throughput, dan konsumsi memori bergantung pada model, input, hardware, kuantisasi, dan pola akses. Angka pada Bagian 35.3 adalah ambang uji, bukan janji untuk setiap kombinasi.

---

## 36. Pengujian dan Verifikasi

### 36.1 Cakupan

- REQ-36-01: Cakupan uji unit MUST ≥ 90% untuk jalur kritis: `nci-core`, `nci-ncl`, `nci-integrity`, `nci-security`, `nci-quant`, `nci-compress`, `nci-storage` (komit dan indeks), dan `nci-routing`. Pengukuran MUST memakai `cargo llvm-cov`.
- REQ-36-02: Cakupan keseluruhan MUST ≥ 80%.
- REQ-36-03: Cakupan dihitung tanpa kode `unsafe` yang dikecualikan.

### 36.2 Jenis Uji

**Uji unit:** encoding varint dan ZigZag, TLV canonical, setiap encoding kolom, setiap format kuantisasi dan pengemasan bit, BLAKE3 dan rumus identitas, state machine reader dan writer, indeks hash.

**Vektor uji tetap:** `conformance/vectors/` MUST berisi:
- Vektor hash untuk setiap domain separator.
- Berkas NCL kecil untuk setiap encoding kolom dan tipe region.
- Berkas NCL yang sengaja rusak (satu bit atau satu byte per kasus) dengan hasil error yang diharapkan.
- Vektor kuantisasi untuk setiap format dengan nilai harapan bit-exact.

**Uji properti:** round-trip (encode lalu decode) untuk kolom, Cell, dan TLV. Dibangkitkan dengan `proptest`.

**Fuzzing:** target fuzz MUST ada untuk: parser superblock, parser region, parser kolom, unpickler, parser safetensors, parser GGUF, parser tokenizer, dan parser TLV. Fuzzing berjalan minimal 1 jam per target sebelum rilis tanpa crash atau timeout.

**Uji integrasi:**
- Konversi penuh: PyTorch, Safetensors, GGUF ke NCL, lalu `check`, lalu `run`.
- Pembacaan selektif dengan pelacakan byte yang dibaca.
- Korupsi: injeksi kerusakan pada blok, kolom, indeks, dan manifest; setiap kasus MUST menghasilkan kode error yang benar dan quarantine.
- Konkurensi: satu writer dan banyak reader selama komit berulang.
- Crash: injeksi kegagalan pada setiap langkah urutan komit (Bagian 15.2).
- Batas memori: uji dengan `RLIMIT_AS` atau cgroup.

**Uji konformansi (`nci selftest`):** binary MUST menyertakan subset vektor uji konformansi dan determinisme untuk pemeriksaan di lapangan.

**Uji kompatibilitas:** berkas v1.0 MUST dapat dibaca oleh pembaca v1.x. Uji lintas versi dijalankan pada setiap rilis.

### 36.3 Pemeriksaan Statis dan Keamanan

- REQ-36-04: CI MUST menjalankan `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo deny check`, `cargo audit`, dan pemeriksaan dependensi runtime (REQ-6-04).
- REQ-36-05: Miri MUST dijalankan pada crate yang mengandung `unsafe`.
- REQ-36-06: Setiap blok `unsafe` MUST ditinjau minimal oleh dua orang.

### 36.4 Uji Skala Model

- REQ-36-07: Model sintetis deterministik (`nci-testkit`) MUST tersedia dengan ukuran 1B, 7B, 13B, 70B, 100B, dan 1T parameter secara logis. Model 1T disimulasikan dengan berkas jarang (sparse) yang diisi pola deterministik agar tidak memerlukan 1 TB penyimpanan fisik di CI.
- REQ-36-08: Model nyata 1B dan 7B MUST diuji pada rig referensi sebelum rilis.

### 36.5 Uji Regresi Akurasi

- REQ-36-09: Setiap kuantisasi MUST diuji terhadap model acuan dengan metrik perplexity pada korpus uji tetap. Batas regresi maksimum 0.5% dibandingkan dengan representasi sumber.

---

## 37. Build, Rilis, dan Distribusi

### 37.1 Artefak

- REQ-37-01: Setiap rilis MUST menghasilkan artefak: binary `nci` untuk setiap target, `SHA256SUMS`, tanda tangan Ed25519 atas `SHA256SUMS`, SBOM (CycloneDX) per target, dan catatan rilis.
- REQ-37-02: Setiap binary MUST dapat diverifikasi dengan `nci version --verify` yang membandingkan hash dirinya dengan nilai yang tertanam dan dengan tanda tangan rilis.
- REQ-37-03: Provenance build SLSA level 3 SEBAIKNYA dihasilkan melalui CI yang terisolasi.

### 37.2 Versi

- REQ-37-04: Versi produk mengikuti SemVer. Versi format NCL terpisah (Bagian 38).
- REQ-37-05: Binary MUST melaporkan versi produk, versi NCL yang didukung, commit, tanggal build, dan target.

### 37.3 Instalasi

- REQ-37-06: Instalasi MUST berupa salin satu berkas ke direktori di PATH. Tidak ada skrip instalasi yang memerlukan akses root.
- REQ-37-07: Tidak ada pembaruan otomatis. Pembaruan dilakukan secara manual oleh pengguna.

### 37.4 Lisensi dan Pihak Ketiga

- REQ-37-08: Lisensi produk ditetapkan oleh pemilik proyek dan dinyatakan di `LICENSE` pada akar repositori.
- REQ-37-09: Setiap dependensi MUST lolos allowlist lisensi di `deny.toml`. Dependensi dengan lisensi tidak kompatibel MUST tidak digunakan.
- REQ-37-10: Notice atribusi dependensi MUST disertakan dalam binary dan dapat diambil dengan `nci version --licenses`.

---

## 38. Kompatibilitas dan Versioning

### 38.1 Versi Format NCL

- REQ-38-01: Versi mayor format NCL berubah jika ada perubahan yang membuat pembaca lama tidak dapat membaca berkas baru.
- REQ-38-02: Versi minor berubah untuk penambahan yang kompatibel mundur (tipe region baru, tag TLV tidak kritis, kode kolom baru yang diabaikan jika tidak dikenal).
- REQ-38-03: Pembaca mayor 1 MUST menolak versi mayor lain dengan `NCI-E-SCHEMA`.
- REQ-38-04: Pembaca minor lebih rendah MUST membaca berkas minor lebih tinggi jika tidak ada bit `feature_flags` yang tidak dikenal yang bersifat kritis. Bit kritis dicadangkan dengan nilai 1 pada bit 31. Jika bit 31 aktif dan tidak dikenal, pembaca MUST menolak.

### 38.2 Perubahan Spesifikasi

- REQ-38-05: Setiap perubahan spesifikasi MUST melalui proses amandemen yang mencatat: alasan, dampak kompatibilitas, versi baru, dan vektor uji yang diperbarui.
- REQ-38-06: Penghapusan fitur MUST diawali penandaan usang (deprecated) minimal satu rilis minor, peringatan saat penggunaan, dan panduan migrasi.
- REQ-38-07: Amandemen yang menambah operator, format kuantisasi, codec, tipe Cell, atau tokenizer MUST memperbarui Bagian 10.2, 17.1, 18, 21.1, atau 21.5 yang relevan.

### 38.3 Migrasi

- REQ-38-08: Perintah `nci migrate <file.ncl>` MUST membaca berkas versi lama dan menulis berkas versi baru ke lokasi terpisah. Operasi ini MUST tidak mengubah berkas asal.

---

## 39. Invarian Tetap

Invarian berikut MUST selalu berlaku dan hanya dapat diubah melalui perubahan mayor spesifikasi.

- **INV-01:** `.ncl` dapat dibaca tanpa pengetahuan format eksternal apapun.
- **INV-02:** Identitas Cell dan payload memakai BLAKE3-256.
- **INV-03:** Identitas Cell diturunkan dari isi canonical dan payload sumber tidak terkompresi.
- **INV-04:** Kompresi tidak mengubah identitas.
- **INV-05:** Pembaca mendukung pembacaan selektif tanpa memuat seluruh berkas.
- **INV-06:** Pembaca mendukung pembacaan selektif kolom.
- **INV-07:** Penulis mendukung input streaming tanpa residensi model penuh.
- **INV-08:** Cell yang sudah terkomit tidak berubah.
- **INV-09:** Payload korup tidak mencapai eksekusi.
- **INV-10:** Ketidakcocokan indeks terdeteksi dan memicu quarantine.
- **INV-11:** Komit bersifat atomik (semua atau tidak sama sekali).
- **INV-12:** Revisi adalah snapshot tak-berubah.
- **INV-13:** Lokasi fisik tidak menentukan identitas.
- **INV-14:** Nama berkas tidak menentukan identitas.
- **INV-15:** Jalur runtime tidak bergantung pada format checkpoint eksternal.
- **INV-16:** Organisasi kolom dipertahankan di setiap lapisan.
- **INV-17:** Binary `nci` adalah satu berkas eksekusi tanpa dependensi runtime eksternal yang wajib.
- **INV-18:** Tidak ada kegagalan senyap; setiap kegagalan menghasilkan kode error dan konteks.
- **INV-19:** Tidak ada perbaikan otomatis terhadap data korup.
- **INV-20:** Tidak ada panggilan jaringan implisit.
- **INV-21:** Tidak ada pelatihan gradien dalam produk.

---

## 40. Kriteria Kesiapan Deployment

Produk dinyatakan siap deployment jika dan hanya jika seluruh kondisi berikut terpenuhi pada commit rilis yang sama:

1. Seluruh persyaratan MUST di dokumen ini terpenuhi, dengan bukti berupa uji lulus atau laporan pemeriksaan.
2. `nci selftest` lulus pada setiap target rilis.
3. Cakupan uji sesuai Bagian 36.1.
4. Seluruh target fuzz berjalan minimal satu jam tanpa temuan baru yang belum diperbaiki.
5. Konversi dan inferensi berhasil untuk setidaknya satu model nyata 1B, satu model nyata 7B, dan satu model sintetis 1T yang lolos kriteria fungsional REQ-35-04.
6. Ambang performa pada Bagian 35.3 telah diisi dan terpenuhi pada rig referensi, atau kegagalannya dicatat sebagai batasan yang diketahui dan disetujui pemilik proyek.
7. Pemeriksaan dependensi runtime (REQ-6-04) lulus.
8. Tidak ada kerentanan terbuka dengan severity tinggi atau kritis pada `cargo audit` atau tinjauan keamanan internal.
9. Tinjauan keamanan independen untuk parser (NCL, PyTorch unpickler, safetensors, GGUF) dan untuk jalur API telah dilakukan, dan temuan kritis telah diperbaiki.
10. Artefak rilis (Bagian 37.1) telah dihasilkan dan diverifikasi.
11. Trust store untuk kunci rilis telah dikonfigurasi dan diuji.
12. Dokumen batasan (Bagian 35.5 dan 29.1) telah disertakan dalam paket rilis.

---

## 41. Penutup

Spesifikasi ini mendefinisikan NCI secara utuh sebagai satu binary `nci` yang membaca dan menulis format `.ncl` yang terdefinisi pada tingkat byte, menjalankan model transformer dengan Cell sebagai unit penyimpanan dan komputasi, mempertahankan memori persisten melalui revisi, dan menyediakan verifikasi integritas serta keamanan yang eksplisit.

Dua hal yang perlu diperhatikan sebelum implementasi dimulai. Pertama, ambang performa pada Bagian 35.3 belum memiliki angka karena bergantung pada rig pengukuran yang belum ditetapkan. Ini harus diisi sebelum kriteria kesiapan deployment dapat diverifikasi, bukan diasumsikan. Kedua, semua keputusan pada Bagian 0.3 mengubah beberapa detail dari spesifikasi v1.0.0 (terutama layout superblock, identitas dua tingkat, dan penghapusan pemanggilan `torch.load`). Jika Anda sudah menulis kode berdasarkan v1.0.0, bagian-bagian tersebut perlu disesuaikan terlebih dahulu.
