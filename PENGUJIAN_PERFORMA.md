# Dokumen Pengujian Performa & Kualitas Knowledge Graph Sanad Hadis

Dokumen ini adalah panduan lengkap untuk 9 jenis pengujian pada sistem **Islamic Knowledge Graph Dashboard**, disusun agar bisa langsung diadaptasi ke **Bab III (Metodologi Penelitian)** dan **Bab IV (Hasil dan Pembahasan)** skripsi.

> ⚠️ **Status per 2026-07-21: seluruh 9 pengujian telah benar-benar dijalankan dan selesai**, termasuk perbandingan before/after index di §8 (§A.8). Sorotan utama: **§9 menemukan kegagalan total sistem di N≈700-800** (§A.9); **§8 menemukan 3 dari 4 query mendapat percepatan 34×-365× setelah indexing, dan membuktikan Bug #1 (name collision) menimpa 15+ nama perawi**, bukan kasus tunggal (§A.8). Hasil di §A **boleh dipakai sebagai data skripsi asli** (bukan ilustrasi) — namun jalankan ulang dengan repetisi lebih besar (20-30x, bukan 6-10x) sebelum final di Bab IV, karena sampel di sini sengaja dipersingkat untuk kecepatan verifikasi. Sisa tabel contoh di seksi §3.1-§3.9 yang masih bertanda `[DATA CONTOH]` (di luar §3.8/3.9 yang sudah diganti data real) adalah ilustrasi format, **bukan** hasil terverifikasi.

---

## A. Ringkasan Eksekusi Nyata (dijalankan 2026-07-21)

Seksi ini adalah hasil sungguhan dari menjalankan skrip-skrip di dokumen ini terhadap dev server lokal (`http://localhost:5174`) dan project Supabase yang aktif dipakai aplikasi. Detail lengkap tiap pengujian tetap ada di §3.1-§3.9; seksi ini adalah ringkasan angka aktual pengganti `[DATA CONTOH]`.

### A.1 Waktu Load Graph — REAL, n=10
```
Percobaan 1:  9.892 ms  (cold start — Vite pre-bundle + koneksi Supabase pertama)
Percobaan 2-10: 1.670 - 2.701 ms
Mean (n=10, termasuk cold start): 2.607 ms | Median: 1.683 ms
Mean (n=9, cold start dikeluarkan): 1.797 ms | Std Dev: 320 ms
```
**Catatan metodologis penting**: percobaan pertama menyimpang jauh (9,9 detik vs ~1,7 detik) karena Vite men-transpile modul untuk pertama kali dan koneksi TCP/TLS ke Supabase belum ada connection reuse. Praktik umum pengujian performa adalah men-*discard* pengukuran *cold-start* pertama dan melaporkannya terpisah — ini sendiri adalah temuan valid untuk dibahas di Bab IV (perbedaan pengalaman pengguna pertama kali vs kunjungan berikutnya).

### A.2 FPS — REAL
```
Fase settling (0-3 dtk pertama):     [62, 61]              rata-rata 61.5, min 61
Idle setelah layout stabil (6 dtk):  [57, 56, 51, 56, 43]   rata-rata 52.6, min 43
Saat pan kanvas:                     [62, 60, 61]           rata-rata 61.0, min 60
Saat drag 1 node:                    [61, 60, 61]           rata-rata 60.7, min 60
```
**Temuan tak terduga**: FPS pada fase "idle stabil" justru lebih rendah (52,6, turun ke 43) dibanding fase "settling" awal (61,5). Ini masuk akal setelah membaca kode: `runTick()` di `GraphCanvas.tsx:249-301` **tidak punya kondisi early-exit/konvergensi** — loop repulsion O(N²) dan spring-force tetap dihitung penuh di setiap frame selamanya, baik posisi node sudah menetap maupun belum. Penurunan ke 43 FPS kemungkinan satu kali *garbage collection pause*; perlu sampel lebih banyak (30+ detik) untuk memastikan ini bukan pola berulang.

### A.3 Penggunaan Memori — REAL
```
Baseline:              8.46 MB
Setelah 2 siklus ganti metrik: 9.15 MB  (+0.69)
Setelah 4 siklus:              9.26 MB  (+0.11)
Setelah 6 siklus:               9.34 MB  (+0.08)
```
Pertumbuhan melandai tajam (+0,69 MB → +0,11 MB → +0,08 MB) — **tidak ada indikasi memory leak**. Konsisten dengan desain `simNodesRef` yang membangun ulang peta simulasi dari `narrators` terkini setiap render.

### A.4 Response Time API Supabase — REAL
| Endpoint | Min | Avg | Median | P95 | P99 | Max |
|---|---|---|---|---|---|---|
| fetchTopNarrators (50) | 177ms | 259ms | 186ms | 351ms | 1.470ms | 1.470ms |
| fetchRelationsForNarrators (query 1/2) | 113ms | 123ms | 120ms | 139ms | 153ms | 153ms |
| fetchEdgeRelations (1000 baris) | 361ms | 413ms | 391ms | 565ms | 828ms | 828ms |
| fetchPerawiTable (50 baris) | 203ms | 268ms | 236ms | 289ms | 1.221ms | 1.221ms |
| fetchAnalyticsData (count) | 175ms | 317ms | 233ms | 502ms | 1.358ms | 1.358ms |

Menarik: `fetchEdgeRelations` (ambil 1000 baris Edge) justru lebih lambat rata-rata (413ms) dibanding `fetchTopNarrators` maupun `fetchRelationsForNarrators`, masuk akal karena volume baris yang ditransfer jauh lebih besar (1000 vs 50). P99 pada beberapa endpoint melonjak ke >1 detik (cold connection pertama pada proses Node yang sama), pola serupa dengan cold-start di §A.1.

### A.5 Load Testing API (k6) — REAL, terhadap REST API Supabase langsung
**Ramping 0→10→50→100→0 VU (95 detik total):**
```
Threshold p(95)<1000ms: GAGAL — p(95) aktual = 4.68s (avg=1.31s, max=12.72s)
Threshold error rate<1%: LOLOS — 0% request gagal (2300/2300 check sukses)
Throughput: 23.9 req/s rata-rata sepanjang ramp
```
**Per level VU tetap (20 detik masing-masing), untuk breakdown yang lebih jelas:**
| VU | Throughput (req/s) | P95 Latency | Max Latency | Error Rate |
|---|---|---|---|---|
| 10 | 12.6 | 595 ms | 1.11 s | 0% |
| 50 | 29.1 | 2.76 s | 7.43 s | 0% |
| 100 | 27.7 | 7.63 s | 15.29 s | 0% |

**Analisis nyata**: sistem **lolos ambang batas 1 detik hanya pada 10 VU bersamaan**. Pada 50 VU, P95 sudah 2,76 detik — melebihi ambang. Pada 100 VU, throughput justru **turun** dari 29,1 menjadi 27,7 req/s dibanding 50 VU sementara latensi meroket ke 7,6 detik — pola klasik *saturasi*: server menerima lebih banyak request daripada yang bisa diproses, request mengantre alih-alih ditolak (error rate tetap 0%, bukan 429 rate-limit), sehingga latensi membengkak alih-alih request gagal. **Breaking point praktis: antara 10-50 pengguna bersamaan** pada tier Supabase yang dipakai saat ini — jauh lebih rendah dari asumsi ilustratif awal (50 VU) di dokumen ini.

### A.6 Graph Completeness — REAL, BUG DITEMUKAN
| Metrik | Node GT | Node App | Edge GT (count exact) | Edge App (diterima) |
|---|---|---|---|---|
| In Degree | 50 | **51** ⚠️ | **1.913** | **1.000** ⚠️ |
| Out Degree | 50 | 50 | — | 1.000 |
| Eigenvector | 50 | **51** ⚠️ | — | 1.000 |
| Closeness | 50 | **51** ⚠️ | — | 1.000 |
| Betweenness | 50 | 50 | — | 1.000 |

**🐛 BUG #1 — Node "bocor" via name collision (bukan bug hipotetis, sudah diverifikasi ada di data nyata):**
`fetchRelationsForNarrators` (`api.ts:80-84`) mencocokkan perawi dengan `.in('Nama_Perawi', names)` — mencari berdasarkan **nama**, bukan `Perawi_ID`. Dibuktikan lewat query:
```sql
SELECT "Nama_Perawi", COUNT(*), array_agg("Perawi_ID")
FROM "Sentralitas" GROUP BY "Nama_Perawi" HAVING COUNT(*) > 1;
```
ditemukan nama **"Anas"** dipakai oleh **dua** `Perawi_ID` berbeda: **12** dan **7404**. Ketika perawi ber-ID 12 masuk top-50 (untuk metrik In-Degree/Eigenvector/Closeness), query nama akan **ikut menarik ID 7404** yang sama sekali bukan bagian dari top-50 — sehingga node dan edge milik "Anas" yang salah bisa tercampur ke dalam graph yang ditampilkan. Ini murni bug identitas (nama tidak unik), bukan keterbatasan desain top-50.
**Rekomendasi perbaikan**: ganti join berbasis nama menjadi berbasis `Perawi_ID` — kirim `Perawi_ID` (bukan `Nama_Perawi`) sebagai kunci dari `fetchTopNarrators` ke `fetchRelationsForNarrators`.

**🐛 BUG #2 — PostgREST "Max Rows" memotong data secara diam-diam (dampak lebih besar):**
Query edge ground-truth (tanpa `.limit()` eksplisit) dan query dengan `.limit(5000)`/`.range(0,1999)` **sama-sama terpotong di 1000 baris**, dikonfirmasi lewat:
```
TRUE total edge count (count:exact,head): 1913
rows actually returned when range(0,1999) requested: 1000
```
Project Supabase ini punya pengaturan **Settings → API → Max Rows** (default PostgREST, umumnya 1000) yang membatasi *setiap* response REST API terlepas dari `.limit()`/`.range()` yang diminta kode — dan tidak memunculkan error, sekadar diam-diam memotong. Akibatnya: **`fetchRelationsForNarrators` kehilangan 913 dari 1913 edge (≈47,7%) untuk metrik In-Degree** saat ini di produksi, tanpa ada indikasi kegagalan di UI.
**Rekomendasi perbaikan**: naikkan "Max Rows" di Supabase Dashboard (Settings → API), atau ubah `fetchRelationsForNarrators`/`fetchEdgeRelations` untuk melakukan paginasi (`.range()` berulang) hingga seluruh baris benar-benar diambil, alih-alih mengandalkan satu `.limit(5000)`.

### A.7 Graph Consistency — REAL
```
[Determinisme] Seluruh 5 metrik: KONSISTEN (5/5 run identik)
[Integritas]   Seluruh 5 metrik: 0 dangling edge, 0 duplikat — TAPI edge count terpotong di 1000 (lihat Bug #2 di atas)
```
Risiko teoretis *non-deterministic tie-breaking* pada `ORDER BY` tanpa secondary sort key (dibahas di §3.7) **tidak terwujud secara empiris** pada dataset saat ini — kemungkinan tidak ada nilai kembar tepat di batas peringkat ke-50 untuk kelima metrik. Namun perlu dicatat: hasil "0 dangling, 0 duplikat" di sini hanya memeriksa 1000 dari seharusnya lebih banyak edge (akibat Bug #2), sehingga cakupan pengujian integritas ini **tidak lengkap** — sekali Bug #2 diperbaiki, uji ulang wajib dilakukan pada seluruh edge.

### A.8 Query Performance — SEBAGIAN REAL (Query A sudah, B/C/D/E menyusul)
Butuh akses langsung ke Postgres (`EXPLAIN ANALYZE`), bukan sekadar anon key, jadi dijalankan manual oleh pengguna via Supabase SQL Editor memakai query siap-pakai di [`scripts/sql/query-analysis.sql`](../scripts/sql/query-analysis.sql).

**Query A (fetchTopNarrators, sebelum index) — REAL:**
```
Limit  (cost=9348.08..9353.83 rows=50 width=128) (actual time=71.477...)
  -> Gather Merge  (cost=9348.08..21324.76 rows=104145 width=128)
       Workers Planned: 1 | Workers Launched: 1
       -> Sort  (Sort Key: "In_Degree_Centrality" DESC, Method: top-N heapsort, Memory: 33kB)
            -> Parallel Seq Scan on "Sentralitas"
Planning Time: 0.464 ms
Execution Time: 76.437 ms
```
**Temuan**: dikonfirmasi `Parallel Seq Scan` (belum ada index pada `In_Degree_Centrality`), PostgreSQL memakai *parallel worker* + *top-N heapsort* sehingga tidak perlu mengurutkan seluruh tabel. **Fakta penting untuk Bab III**: `rows=104145` berarti tabel `Sentralitas` berisi **~104.145 baris perawi** — total dataset jauh lebih besar dari 50 node yang divisualisasikan. Angka 76,4 ms ini jadi baseline "sebelum index"; belum ada angka "sesudah index" karena `CREATE INDEX` di baris 44-56 file SQL belum dijalankan.

**Query B (fetchRelationsForNarrators tahap 2, Edge double-IN, sebelum index) — REAL:**
```
Gather  (cost=1000.00..69487.40 rows=3265 width=24) (actual time=3.8...)
  Workers Planned: 1 | Workers Launched: 1
  -> Parallel Seq Scan on "Edge"
       Filter: (("Guru_Id" = ANY ('{1,2,3,...50 id...}')) AND ("Murid_Id" = ANY (...)))
       Rows Removed by Filter: 387442
Planning Time: 0.590 ms
Execution Time: 1228.484 ms
```
**Temuan**: `Parallel Seq Scan` (belum ada index pada `Guru_Id`/`Murid_Id`), **1.228 ms** — jauh lebih lambat dari Query A. `Rows Removed by Filter: 387.442` mengonfirmasi tabel `Edge` berisi **~389.000 baris total** (387.442 + ±1.913 baris yang cocok, angka 1.913 ini sama persis dengan ground-truth count yang sudah diverifikasi di §A.6). Filter ganda pada dua kolom tanpa index berarti PostgreSQL memeriksa hampir seluruh tabel baris demi baris untuk query yang seharusnya hanya butuh ~1.900 baris — kandidat index paling kuat dari seluruh pengujian ini.

**Query C (searchPerawiByName, ILIKE, sebelum index) — REAL:**
```
Limit  (cost=0.00..169.44 rows=200 width=8) (actual time=0.029..6.48...)
  -> Seq Scan on "Sentralitas"
       Filter: ("Nama_Perawi" ~~* '%ibn%'::text)
       Rows Removed by Filter: 4660
Planning Time: 3.730 ms
Execution Time: 6.603 ms
```
**Temuan**: `Seq Scan` dikonfirmasi (ILIKE dengan wildcard di awal tidak bisa pakai B-tree index), tapi eksekusinya cepat (**6,6 ms**) karena query ini **tidak punya `ORDER BY`** — hanya `LIMIT 200`. PostgreSQL berhenti scan begitu 200 baris cocok ditemukan (`Rows Removed by Filter: 4.660` jauh lebih kecil dari total ~104.145 baris tabel), bukan memindai seluruh tabel. **Catatan penting untuk analisis**: kecepatan ini adalah kebetulan akibat kata kunci `'ibn'` cukup sering muncul di awal-awal urutan fisik tabel; kata kunci yang lebih jarang berpotensi memaksa scan hampir penuh sebelum `LIMIT` terpenuhi, sehingga performa nyatanya *tidak stabil* — bervariasi tergantung isi pencarian, bukan cuma soal ada/tidaknya index.

**Query D (fetchNarratorCounts, count exact, sebelum index) — REAL:**
```
Finalize Aggregate (actual time=...)
  -> Gather
       Workers Planned: 1 | Workers Launched: 1
       -> Partial Aggregate
            -> Parallel Seq Scan on "Edge"
                 Filter: ("Guru_Id" = 12)
                 Rows Removed by Filter: 387672
Planning Time: 0.280 ms
Execution Time: 846.632 ms
```
**Temuan**: `Parallel Seq Scan`, **846,6 ms** untuk sekadar menghitung baris dengan satu nilai `Guru_Id`. Konsisten dengan Query B — tabel `Edge` (~389 ribu baris) di-scan penuh karena tidak ada index pada `Guru_Id`. (Sebagai konteks: `Perawi_ID` 12 adalah "Anas", perawi peringkat #1 In-Degree Centrality — masuk akal jika ia punya banyak baris `Guru_Id` yang cocok, karena ia paling banyak dirujuk sebagai guru.)

**Sesudah `CREATE INDEX` dijalankan — REAL (perbandingan before/after lengkap):**

| Query | Sebelum Index | Sesudah Index | Peningkatan |
|---|---|---|---|
| A: Top-50 by In_Degree | 76,437 ms, Parallel Seq Scan + top-N heapsort | **1,435 ms**, `Index Scan using idx_sentralitas_in_degree` | **~53×** |
| B: Edge double-IN | 1.228,484 ms, Parallel Seq Scan (±389rb baris) | **35,639 ms**, `BitmapAnd` menggabungkan `idx_edge_murid` + `idx_edge_guru` | **~34×** |
| C: ILIKE search | 6,603 ms, Seq Scan (early-exit `LIMIT` tanpa `ORDER BY`) | **6,751 ms**, `Bitmap Index Scan` pada `idx_sentralitas_nama_trgm` | **~tidak berubah (lihat catatan)** |
| D: Count by Guru_Id | 846,632 ms, Parallel Seq Scan (±389rb baris) | **2,322 ms**, `Index Only Scan using idx_edge_guru` (Heap Fetches: 184) | **~365×** |

**Analisis per query:**
- **Query A**: rencana berubah total dari `Parallel Seq Scan + top-N heapsort` menjadi `Index Scan` langsung pada `idx_sentralitas_in_degree` — PostgreSQL tinggal membaca 50 baris pertama dari index yang sudah terurut, tanpa perlu memindai atau mengurutkan apa pun. Planning time naik (0,464→7,102 ms, wajar karena planner mengevaluasi index baru untuk pertama kali) tapi execution time turun drastis.
- **Query B**: PostgreSQL memilih `BitmapAnd` — menjalankan **dua** index scan terpisah (`idx_edge_murid` dan `idx_edge_guru`), lalu mengiriskan (AND) hasil kedua bitmap sebelum mengambil baris tabel yang relevan (`Heap Blocks: exact=1196`). Ini pola optimal untuk filter ganda pada dua kolom yang masing-masing punya index sendiri.
- **Query C — tidak membaik, dan ini valid, bukan anomali**: karena baseline "sebelum index" *sudah* cepat akibat `LIMIT` tanpa `ORDER BY` (early-exit begitu 200 baris ditemukan — lihat catatan di hasil Query C sebelum-index), index GIN trigram tidak punya banyak ruang untuk memberi manfaat pada kasus spesifik kata kunci `'ibn'` yang umum. Index ini akan menunjukkan manfaat jauh lebih besar untuk kata kunci yang **lebih jarang muncul** (di mana seq scan tanpa index harus memindai jauh lebih banyak baris sebelum kuota `LIMIT` terpenuhi) — sebuah skenario yang tidak tercakup oleh pengujian dengan kata kunci `'ibn'` saja.
- **Query D — peningkatan paling dramatis**: berubah menjadi `Index Only Scan` — PostgreSQL bisa menjawab query **tanpa membuka tabel heap sama sekali** dalam banyak kasus (hanya 184 "Heap Fetches" untuk baris yang belum ter-*visibility-map*), karena semua data yang dibutuhkan (`Guru_Id` untuk keperluan `COUNT`) sudah tersedia langsung di index itu sendiri.

**Kesimpulan gabungan**: 3 dari 4 query mengalami percepatan 34×-365×; satu query (C) menunjukkan bahwa index bukan solusi universal — dampaknya bergantung pada apakah query sebelumnya sudah kebetulan diuntungkan oleh optimasi lain (di sini, *early-exit* `LIMIT`). Ini nuansa penting untuk dituliskan di Bab IV agar analisis tidak terkesan "index selalu menang tanpa syarat".

**Query E (bukti Bug #1, name collision) — REAL, jauh lebih parah dari perkiraan awal:**
Hasil `GROUP BY "Nama_Perawi" HAVING COUNT(*) > 1` menampilkan **belasan nama kembar**, bukan cuma "Anas" seperti temuan awal saya:
```
Hban                                          -> 2 baris (ID: 1083, 11147...)
Bisyr bin Syhan                               -> 2 baris (ID: 7458, 13228...)
Abu Ja'far Muhammad bin Ali bin Dhym Al-Syybany -> 2 baris (ID: 2596, 16475...)
Dhtsm                                         -> 2 baris
Tby' Al-Hjry                                  -> 2 baris
Muhammad bin Haman Al-Jndysabwry              -> 2 baris
Hbyb                                          -> 2 baris
Muhammad bin Ibrahim Al-Hrwy                  -> 2 baris
Hmdan                                         -> 2 baris
Ibrahim bin Hrym                              -> 2 baris
Al-Dhhak Al-Hmdany                            -> 2 baris
Frat Al-Bhrany                                -> 2 baris
Al-Mkhwl                                      -> 2 baris
Abdullah bin Sybwyh                           -> 2 baris
Abu Anas                                      -> 2 baris
... (daftar berlanjut, terpotong pada tangkapan layar)
```
**Revisi tingkat keparahan Bug #1**: temuan sebelumnya di §A.6 ("Anas", ID 12 vs 7404) hanya **satu contoh** dari masalah yang jauh lebih luas. Setidaknya 15 nama berbeda punya duplikasi `Perawi_ID` di tabel `Sentralitas` — artinya **setiap kali salah satu dari nama-nama ini masuk daftar top-N**, `fetchRelationsForNarrators` (yang men-join lewat `Nama_Perawi`, bukan `Perawi_ID`) berisiko menarik data perawi yang salah. Ini menaikkan status Bug #1 dari "ditemukan satu kasus" menjadi **cacat desain sistemik pada strategi join** yang harus diperbaiki di level kode (ganti ke join berbasis ID), bukan sekadar dibersihkan datanya (karena nama duplikat kemungkinan akan terus muncul selama identitas asli memakai `Perawi_ID`, bukan nama).

### A.9 Scalability Testing — REAL, dan menemukan titik kegagalan keras (bukan cuma degradasi bertahap)
Parameterisasi N (`?n=` di URL) diterapkan ke `useGraphData.ts` sesuai diff di §3.9 (default tetap 50, hanya aktif bila parameter diberikan). Playwright dijalankan untuk N = 50, 100, 250, 500, 1000.

| N (diminta) | Node diterima | Edge diterima (Stat Card) | Load Time | FPS Rata-rata | FPS Min | Heap |
|---|---|---|---|---|---|---|
| 50 | 50 | 1.000 (capped, lihat Bug #2) | 9.407 ms *(cold start ulang)* | 59,8 | 58 | 8,8 MB |
| 100 | 100 | 1.000 (capped) | 2.707 ms | 61,0 | 61 | 11,1 MB |
| 250 | 250 | 1.000 (capped) | 2.788 ms | 59,3 | 54 | 10,1 MB |
| 500 | 500 | 1.000 (capped) | 3.263 ms | 56,8 | 53 | 10,3 MB |
| 1000 | **GAGAL TOTAL** | — | **timeout >45.000 ms** | — | — | — |

**Temuan tak terduga #1 — FPS jauh lebih baik dari dugaan**: dari N=50 sampai N=500, FPS tetap di kisaran 57-61, TIDAK jatuh ke puluhan seperti perkiraan ilustratif awal di §3.9. Artinya loop repulsion O(N²) di `GraphCanvas.tsx` **bukan bottleneck nyata** pada rentang ini — 500² = 250.000 pasangan per frame ternyata masih ringan untuk V8 dalam anggaran 16ms/frame. Kesimpulan awal di dokumen ini (yang menduga rendering jadi bottleneck utama) **terkoreksi oleh data nyata**.

**Temuan tak terduga #2 — bottleneck sebenarnya ada di layer fetch data, dan berbentuk tebing, bukan lereng**: pengujian N=1000 gagal total (Playwright timeout 45 detik, spinner loading tidak pernah hilang). Investigasi langsung ke Supabase (di luar browser) mengonfirmasi akar masalahnya:
```
fetchTopNarrators(1000): 750 ms — OK
name-lookup .in('Nama_Perawi', 1000 nama): TypeError: fetch failed
```
Fungsi `fetchRelationsForNarrators` (`api.ts:80-84`) mem-*build* filter `.in('Nama_Perawi', names)` yang di-encode sebagai query-string URL. Panjang string ini linear terhadap N:
| Jumlah nama di filter | Panjang query-string (ter-encode) | Hasil |
|---|---|---|
| 600 | ~11.600 karakter | ✅ OK (397 ms) |
| 700 | ~13.600 karakter | ✅ OK (242 ms) |
| 800 | ~15.500 karakter | ❌ **`fetch failed`** |
| 1000 | 19.408 karakter (diukur persis) | ❌ **`fetch failed`** |

**Ambang kegagalan nyata berada di antara N=700 dan N=800** (bergantung panjang nama masing-masing perawi, jadi bukan angka N yang tetap, tapi fungsi dari total karakter ter-encode) — kemungkinan besar menabrak batas ukuran URL/header default (banyak infrastruktur HTTP membatasi sekitar 8-16KB). Ini **bukan degradasi bertahap seperti FPS, tapi kegagalan biner**: di bawah ambang sistem bekerja normal, di atasnya permintaan gagal total dan UI macet permanen di layar loading (tidak ada pesan error yang muncul ke pengguna — `useGraphData.ts` menangkap error lewat `.catch()` tapi pada kasus ini promise tampaknya tergantung, bukan reject bersih, sehingga `finally()` juga tidak sempat mengeksekusi dalam window pengamatan).

**Implikasi untuk rencana "mencoba node lain" (pertanyaan awal Anda)**: aman menaikkan N hingga sekitar 500-600. **Jangan naikkan ke atas ~700-800 tanpa memperbaiki arsitektur query terlebih dahulu** — bukan soal performa lagi di titik itu, tapi sistem benar-benar berhenti berfungsi. Perbaikan yang direkomendasikan (sekaligus juga memperbaiki Bug #1 dan Bug #2 di §A.6): ganti pendekatan 2-3 *round-trip* dengan filter `.in()` besar menjadi satu **Postgres function/RPC** yang melakukan JOIN `Sentralitas`+`Edge` di sisi database — menghilangkan ketergantungan pada panjang URL sekaligus pada cap "Max Rows" PostgREST.

---

## 0. Catatan Penting: Kesesuaian Teknologi

Berdasarkan pemeriksaan kode aktual (per 2026-07-21), stack sistem ini adalah:

| Klaim umum | Realita di kode |
|---|---|
| Cytoscape.js | ❌ Tidak ditemukan (`cytoscape` tidak ada di `package.json`, tidak ada import di manapun) |
| Rendering graph | ✅ **Custom Canvas 2D force-directed engine** — [`src/app/components/GraphCanvas.tsx`](../src/app/components/GraphCanvas.tsx), pakai `<canvas>` mentah + `requestAnimationFrame` + simulasi fisika manual (repulsion antar-node, spring pada edge, constraint radial per-ring) |
| React | ✅ React 18.3.1 + React Router 7 + TypeScript + Vite 6 |
| Supabase | ✅ `@supabase/supabase-js` 2.108.1, tabel `Sentralitas` (node/perawi) dan `Edge` (relasi guru–murid) |
| Styling | Tailwind CSS 4 |

**Rekomendasi**: di Bab I/III, sebut komponen visualisasi sebagai *"mesin rendering graph berbasis HTML5 Canvas dengan algoritma force-directed custom"*, bukan Cytoscape.js. Ini justru jadi nilai tambah metodologis karena Anda bisa membahas trade-off *build vs. use library* di Bab V (saran: migrasi ke Cytoscape.js/D3-force sebagai pengembangan lanjutan, dengan data performa di dokumen ini sebagai baseline pembanding).

**Batasan arsitektur yang relevan untuk pengujian** (ditemukan di kode, akan dirujuk di seksi-seksi berikut):
- `useGraphData.ts:23` — jumlah node **hardcoded ke 50**: `fetchTopNarrators(selectedMetric, 50)`. Untuk pengujian skalabilitas (§9), nilai ini perlu diparameterisasi.
- `GraphCanvas.tsx:253-266` — loop repulsion antar-node bersarang **O(N²)** per frame. Ini adalah kandidat bottleneck utama saat N membesar.
- `api.ts:54-74` (`fetchTopNarrators`) — `ORDER BY <kolom> DESC LIMIT 50` **tanpa secondary sort key**, berpotensi non-deterministik saat ada nilai kembar (dibahas di §7).
- `api.ts:76-122` (`fetchRelationsForNarrators`) — 2 round-trip berurutan (query `Sentralitas` lalu `Edge`), relevan untuk §4 dan §8.

---

## 1. Persiapan Lingkungan Pengujian

Instal tools berikut sekali di awal (jalankan dari root folder proyek):

```powershell
# Playwright — untuk load time, FPS, memory (browser otomatis)
npm install -D @playwright/test
npx playwright install chromium

# dotenv — agar script Node bisa baca .env
npm install -D dotenv

# k6 — untuk load testing API (binary terpisah, bukan npm package)
choco install k6
# atau unduh manual: https://github.com/grafana/k6/releases
```

Buat struktur folder pengujian (terpisah dari source aplikasi, tidak mengubah kode produksi):

```
tests/performance/     -> spesifikasi Playwright (load time, FPS, memory, scalability)
scripts/load-test/     -> skrip k6
scripts/analysis/      -> skrip Node.js (response time, completeness, consistency)
scripts/sql/           -> query EXPLAIN ANALYZE untuk Supabase SQL Editor
```

Jalankan dev server sebelum pengujian browser:

```powershell
npm run dev   # default: http://localhost:5173
```

---

## 2. Kerangka Acuan (untuk Bab III)

Gunakan **ISO/IEC 25010** (Software Product Quality Model) sebagai kerangka teoretis pengujian non-fungsional:

| Karakteristik ISO 25010 | Pengujian di dokumen ini |
|---|---|
| Performance Efficiency (Time Behaviour) | §1 Waktu Load, §2 FPS, §4 Response Time, §8 Query Performance |
| Performance Efficiency (Resource Utilization) | §3 Penggunaan Memori |
| Performance Efficiency (Capacity) | §5 Load Testing, §9 Scalability Testing |
| Functional Suitability (Completeness, Correctness) | §6 Graph Completeness, §7 Graph Consistency |

Ambang batas rujukan yang dipakai di seksi analisis:
- **Waktu respons** — Nielsen (1993): <0.1 dtk = instan, <1 dtk = alur tidak terputus, <10 dtk = batas atensi pengguna.
- **Frame rate** — web.dev/Google: 60 FPS = mulus, 30 FPS = batas minimum dapat diterima, <30 FPS = *jank* terasa.
- **API latency** — Google RAIL model: <100ms untuk respons terasa instan; <1000ms umum dianggap batas atas UX yang baik untuk request async.

---

# 3. Sembilan Pengujian

## 3.1 Waktu Load Graph

### Tujuan
Mengukur waktu dari user membuka halaman Knowledge Graph Explorer hingga graph (50 node) selesai dimuat dan stabil tervisualisasi di layar.

### Alat yang Digunakan
- **Playwright** (otomasi + pengukuran presisi)
- **Chrome DevTools → Performance panel** (verifikasi manual/visual)
- **Web Performance API** (`performance.now()`, `performance.mark`)

### Langkah-Langkah Pengujian
1. Jalankan `npm run dev`, pastikan aplikasi bisa diakses di `http://localhost:5173`.
2. Definisikan titik akhir pengukuran: teks *"Memuat data dari Database..."* (spinner loading, lihat `KnowledgeGraphExplorer.tsx:112`) hilang dari DOM = data selesai fetch.
3. Ulangi pengukuran **20 kali** (percobaan independen, cold cache tiap kali via `page.goto` baru) untuk validitas statistik.
4. Ulangi pada 3 kondisi jaringan: **No Throttling**, **Fast 3G**, **Slow 3G** (via `page.route`/CDP network emulation) untuk melihat sensitivitas terhadap jaringan.
5. Hitung mean, median, std. deviation, dan persentil 95.

### Kode yang Diperlukan

```ts
// tests/performance/load-time.spec.ts
import { test } from '@playwright/test';

const REPEATS = 20;
const results: number[] = [];

test.describe.serial('Waktu Load Graph (50 node)', () => {
  for (let i = 0; i < REPEATS; i++) {
    test(`percobaan-${i + 1}`, async ({ page }) => {
      const t0 = performance.now();
      await page.goto('http://localhost:5173/', { waitUntil: 'domcontentloaded' });

      // titik akhir: spinner "Memuat data dari Database..." hilang dari DOM
      await page.waitForSelector('text=Memuat data dari Database', {
        state: 'detached',
        timeout: 30000,
      });
      const t1 = performance.now();

      const duration = Math.round(t1 - t0);
      results.push(duration);
      console.log(`Percobaan ${i + 1}: ${duration} ms`);
    });
  }

  test.afterAll(() => {
    const sorted = [...results].sort((a, b) => a - b);
    const mean = results.reduce((a, b) => a + b, 0) / results.length;
    const median = sorted[Math.floor(sorted.length / 2)];
    const p95 = sorted[Math.floor(sorted.length * 0.95)];
    const stdDev = Math.sqrt(
      results.reduce((s, v) => s + (v - mean) ** 2, 0) / results.length
    );
    console.log(`\n=== Ringkasan Waktu Load ===`);
    console.log(`Mean: ${mean.toFixed(1)} ms | Median: ${median} ms`);
    console.log(`Std Dev: ${stdDev.toFixed(1)} ms | P95: ${p95} ms`);
    console.log(`Min: ${sorted[0]} ms | Max: ${sorted[sorted.length - 1]} ms`);
  });
});
```

```ts
// playwright.config.ts (untuk emulasi jaringan, dijalankan per-proyek)
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  projects: [
    { name: 'no-throttle', use: { ...devices['Desktop Chrome'] } },
    {
      name: 'fast-3g',
      use: {
        ...devices['Desktop Chrome'],
        launchOptions: { args: [] },
        // gunakan CDP throttling via test.beforeEach jika perlu presisi lebih
      },
    },
  ],
});
```

Jalankan: `npx playwright test tests/performance/load-time.spec.ts --project=no-throttle`

### Parameter yang Diukur
| Parameter | Satuan | Definisi |
|---|---|---|
| Time to Data Fetch | ms | Waktu request Supabase pertama hingga data diterima |
| Time to First Render | ms | Waktu hingga node pertama tergambar di canvas |
| Time to Stable Layout | ms | Waktu hingga simulasi fisika + auto-fit selesai (≈2.5 dtk setelah data tersedia, lihat `GraphCanvas.tsx:588-592`) |
| Total Load Time | ms | `page.goto` → spinner hilang |

### Contoh Tabel Hasil
| Percobaan | No Throttle (ms) | Fast 3G (ms) | Slow 3G (ms) |
|---|---|---|---|
| 1 | 812 | 1.940 | 4.310 |
| 2 | 795 | 1.885 | 4.402 |
| ... | ... | ... | ... |
| 20 | 830 | 2.010 | 4.275 |
| **Mean** | **[DATA CONTOH] 815** | **[DATA CONTOH] 1.930** | **[DATA CONTOH] 4.340** |
| **Median** | [DATA CONTOH] 810 | [DATA CONTOH] 1.920 | [DATA CONTOH] 4.320 |
| **Std Dev** | [DATA CONTOH] 18 | [DATA CONTOH] 45 | [DATA CONTOH] 60 |
| **P95** | [DATA CONTOH] 845 | [DATA CONTOH] 2.005 | [DATA CONTOH] 4.410 |

### Cara Analisis Hasil
1. Bandingkan mean vs median — jika jauh berbeda, ada outlier (periksa apakah cold-start Supabase/koneksi pertama menyebabkan lonjakan).
2. Bandingkan std dev antar kondisi jaringan — std dev besar pada Slow 3G wajar (bottleneck jaringan lebih dominan dari komputasi).
3. Uji terhadap ambang Nielsen: apakah median < 1000ms (ideal), atau masuk kategori "masih dapat diterima" (<10 dtk)?
4. Pisahkan kontribusi *fetch* vs *render*: jika fetch mendominasi, optimasi difokuskan ke query (§8); jika render mendominasi, optimasi difokuskan ke algoritma layout (§9).

### Contoh Penulisan Bab III
> Pengujian waktu load graph dilakukan secara otomatis menggunakan framework Playwright untuk mengukur interval waktu antara permintaan halaman (`page.goto`) hingga indikator pemuatan data hilang dari DOM, yang menandakan proses *fetching* data dari Supabase dan render awal 50 node telah selesai. Pengujian diulang sebanyak 20 kali pada tiga kondisi jaringan (tanpa *throttling*, Fast 3G, dan Slow 3G) untuk memperoleh distribusi waktu yang representatif. Dari setiap set pengujian dihitung nilai rata-rata, median, standar deviasi, dan persentil ke-95 sebagai indikator stabilitas performa.

### Contoh Penulisan Bab IV
> Hasil pengujian waktu load graph pada kondisi jaringan tanpa *throttling* menunjukkan rata-rata waktu pemuatan sebesar **[DATA CONTOH] 815 ms** dengan standar deviasi **18 ms**, mengindikasikan performa yang konsisten antar percobaan. Nilai ini berada di bawah ambang batas 1 detik yang menurut Nielsen (1993) merepresentasikan batas di mana pengguna masih merasakan alur interaksi yang tidak terputus. Pada simulasi jaringan Slow 3G, waktu load meningkat signifikan menjadi rata-rata **[DATA CONTOH] 4.340 ms**, menunjukkan bahwa latensi jaringan — bukan kompleksitas komputasi *client-side* — menjadi faktor dominan pada kondisi konektivitas terbatas.

---

## 3.2 Frame Per Second (FPS)

### Tujuan
Mengukur kelancaran animasi rendering canvas selama simulasi fisika berjalan dan saat pengguna berinteraksi (pan, zoom, drag node), mengingat rendering memakai *custom* RAF loop (`GraphCanvas.tsx:437-444`), bukan engine WebGL/Cytoscape yang punya optimasi bawaan.

### Alat yang Digunakan
- **Playwright** dengan skrip injeksi `requestAnimationFrame` counter (tidak perlu ubah kode produksi)
- **Chrome DevTools** → `Ctrl+Shift+P` → *"Show frames per second (FPS) meter"* (verifikasi visual)

### Langkah-Langkah Pengujian
1. Buka graph, tunggu data selesai dimuat.
2. Ukur FPS pada 4 skenario: **(a)** idle 2,5 detik pertama saat simulasi fisika masih menata posisi, **(b)** idle setelah layout stabil, **(c)** saat *pan* (drag kanvas kosong), **(d)** saat *drag* satu node.
3. Setiap skenario direkam selama 5–8 detik, FPS disampel tiap 1 detik.
4. Ulangi seluruh skenario 10 kali, ambil rata-rata dan minimum per skenario.

### Kode yang Diperlukan

```ts
// tests/performance/fps.spec.ts
import { test, type Page } from '@playwright/test';

async function measureFps(page: Page, durationMs: number): Promise<number[]> {
  return page.evaluate((duration) => {
    return new Promise<number[]>((resolve) => {
      const samples: number[] = [];
      let frames = 0;
      let lastSecond = performance.now();
      const start = performance.now();

      function tick() {
        frames++;
        const now = performance.now();
        if (now - lastSecond >= 1000) {
          samples.push(frames);
          frames = 0;
          lastSecond = now;
        }
        if (now - start < duration) requestAnimationFrame(tick);
        else resolve(samples);
      }
      requestAnimationFrame(tick);
    });
  }, durationMs);
}

test('FPS - idle setelah layout stabil', async ({ page }) => {
  await page.goto('http://localhost:5173/');
  await page.waitForSelector('text=Memuat data dari Database', { state: 'detached' });
  await page.waitForTimeout(3500); // lewati auto-fit (2.5 dtk) agar simulasi sudah stabil
  const samples = await measureFps(page, 6000);
  console.log('FPS/detik:', samples);
  console.log('Rata-rata:', (samples.reduce((a, b) => a + b, 0) / samples.length).toFixed(1));
  console.log('Minimum:', Math.min(...samples));
});

test('FPS - saat pan kanvas', async ({ page }) => {
  await page.goto('http://localhost:5173/');
  await page.waitForSelector('text=Memuat data dari Database', { state: 'detached' });
  await page.waitForTimeout(3500);

  const box = await page.locator('canvas').boundingBox();
  if (!box) throw new Error('Canvas tidak ditemukan');

  const fpsPromise = measureFps(page, 4000);
  await page.mouse.move(box.x + box.width / 2, box.y + box.height / 2);
  await page.mouse.down();
  for (let i = 0; i < 40; i++) {
    await page.mouse.move(box.x + box.width / 2 + i * 3, box.y + box.height / 2 + i * 1.5);
    await page.waitForTimeout(25);
  }
  await page.mouse.up();
  const samples = await fpsPromise;
  console.log('FPS saat pan:', samples);
});

test('FPS - saat drag satu node', async ({ page }) => {
  await page.goto('http://localhost:5173/');
  await page.waitForSelector('text=Memuat data dari Database', { state: 'detached' });
  await page.waitForTimeout(3500);

  const box = await page.locator('canvas').boundingBox();
  if (!box) throw new Error('Canvas tidak ditemukan');
  const centerX = box.x + box.width / 2;
  const centerY = box.y + box.height / 2;

  const fpsPromise = measureFps(page, 4000);
  await page.mouse.move(centerX, centerY); // node rank-1 biasanya di tengah (radius ring = 0)
  await page.mouse.down();
  for (let i = 0; i < 40; i++) {
    await page.mouse.move(centerX + Math.sin(i / 5) * 80, centerY + Math.cos(i / 5) * 80);
    await page.waitForTimeout(25);
  }
  await page.mouse.up();
  console.log('FPS saat drag node:', await fpsPromise);
});
```

### Parameter yang Diukur
| Parameter | Satuan | Keterangan |
|---|---|---|
| FPS rata-rata | fps | Rata-rata seluruh sampel per detik |
| FPS minimum | fps | Nilai terendah (indikator *jank* terparah) |
| % frame di bawah 30 FPS | % | Proporsi detik dengan FPS < 30 |
| % frame di bawah 60 FPS | % | Proporsi detik dengan FPS < 60 |

### Contoh Tabel Hasil
| Skenario | FPS Rata-rata | FPS Min | % < 30 FPS | Kategori |
|---|---|---|---|---|
| Idle (layout stabil) | [DATA CONTOH] 59.4 | 57 | 0% | Mulus |
| Idle (fase settling 0–2.5s) | [DATA CONTOH] 52.1 | 41 | 5% | Mulus |
| Pan kanvas | [DATA CONTOH] 58.2 | 50 | 0% | Mulus |
| Drag 1 node | [DATA CONTOH] 56.7 | 47 | 0% | Mulus |

### Cara Analisis Hasil
1. Bandingkan terhadap ambang 60 FPS (ideal) dan 30 FPS (batas minimum, web.dev).
2. Fase *settling* (0–2.5 dtk) diperkirakan lebih berat karena loop repulsion `O(N²)` di `GraphCanvas.tsx:253-266` bekerja penuh saat node masih saling menjauh dari posisi awal acak — jika FPS di fase ini jauh lebih rendah dari fase idle, ini mengonfirmasi repulsion sebagai kontributor beban komputasi utama.
3. Jadikan hasil §3.2 sebagai baseline sebelum pengujian skalabilitas (§3.9) — FPS pada N=50 adalah titik acuan untuk melihat penurunan performa saat N naik.

### Contoh Penulisan Bab III
> Pengukuran *frame rate* dilakukan dengan menyisipkan penghitung `requestAnimationFrame` melalui `page.evaluate()` pada Playwright, tanpa memodifikasi kode aplikasi. FPS disampel setiap detik selama jendela pengamatan 4–6 detik pada empat skenario: kondisi diam pada fase penataan awal simulasi fisika, kondisi diam setelah layout stabil, interaksi *pan* kanvas, dan interaksi *drag* node tunggal. Setiap skenario diulang 10 kali untuk memperoleh nilai rata-rata dan nilai minimum sebagai indikator *jank*.

### Contoh Penulisan Bab IV
> Pengujian FPS pada kondisi idle setelah layout stabil menunjukkan rata-rata **[DATA CONTOH] 59,4 FPS** dengan nilai minimum 57 FPS, mendekati batas ideal 60 FPS untuk animasi berbasis Canvas 2D. Namun, pada fase *settling* awal (0–2,5 detik pertama setelah data dimuat), FPS rata-rata turun menjadi **[DATA CONTOH] 52,1 FPS** dengan nilai minimum mencapai 41 FPS. Penurunan ini konsisten dengan struktur algoritma repulsion antar-node yang bersifat kuadratik (O(N²)) pada `GraphCanvas.tsx`, yang bebannya paling tinggi ketika node masih tersebar jauh dari posisi target sebelum konvergen ke layout ring berbasis peringkat sentralitas.

---

## 3.3 Penggunaan Memori Browser

### Tujuan
Mengukur konsumsi JS heap saat graph dimuat dan setelah interaksi berulang (ganti metrik sentralitas, pilih/batal-pilih node), untuk mendeteksi potensi *memory leak* — khususnya karena `useGraphData` melakukan *refetch* penuh setiap `selectedMetric` berubah (`useGraphData.ts:32`) dan `GraphCanvas` merekonstruksi `simNodesRef` (`GraphCanvas.tsx:200-223`).

### Alat yang Digunakan
- **Chrome DevTools → Memory tab** (Heap Snapshot, verifikasi manual)
- **Playwright + CDP `Performance.getMetrics`** (otomasi, `JSHeapUsedSize`)

### Langkah-Langkah Pengujian
1. Baseline: ambil heap snapshot segera setelah graph pertama kali stabil.
2. Lakukan 10 siklus interaksi: klik metrik "Out Degree Centrality" → klik metrik "In Degree Centrality" (kembali ke awal), memicu refetch + rebuild simulasi tiap siklus.
3. Ambil heap snapshot setelah setiap 2 siklus (5 snapshot total).
4. Lakukan juga 10 siklus pilih-node lalu batal-pilih (klik node → klik node yang sama untuk deselect).
5. Bandingkan tren heap size antar snapshot — pertumbuhan linear tanpa turun kembali setelah *forced GC* mengindikasikan leak.

### Kode yang Diperlukan

```ts
// tests/performance/memory.spec.ts
import { test } from '@playwright/test';

test('Memory - siklus ganti metrik berulang', async ({ page }) => {
  const client = await page.context().newCDPSession(page);
  await page.goto('http://localhost:5173/');
  await page.waitForSelector('text=Memuat data dari Database', { state: 'detached' });
  await page.waitForTimeout(3000);

  async function heapMB() {
    await client.send('HeapProfiler.collectGarbage'); // paksa GC sebelum ukur
    const { metrics } = await client.send('Performance.getMetrics');
    const heap = metrics.find(m => m.name === 'JSHeapUsedSize')!.value;
    return (heap / 1024 / 1024).toFixed(2);
  }

  console.log(`Baseline: ${await heapMB()} MB`);

  for (let cycle = 1; cycle <= 10; cycle++) {
    await page.getByRole('button', { name: 'Out Degree Centrality' }).click();
    await page.waitForSelector('text=Memuat data dari Database', { state: 'detached' });
    await page.waitForTimeout(500);
    await page.getByRole('button', { name: 'In Degree Centrality' }).click();
    await page.waitForSelector('text=Memuat data dari Database', { state: 'detached' });
    await page.waitForTimeout(500);

    if (cycle % 2 === 0) {
      console.log(`Setelah siklus ${cycle}: ${await heapMB()} MB`);
    }
  }
});
```

### Parameter yang Diukur
| Parameter | Satuan | Keterangan |
|---|---|---|
| JS Heap Used | MB | `performance.memory.usedJSHeapSize` / CDP `JSHeapUsedSize` |
| Pertumbuhan heap per siklus | MB/siklus | Selisih heap antar snapshot |
| Jumlah Detached DOM Nodes | node | Dari DevTools Memory → Heap Snapshot → filter "Detached" |

### Contoh Tabel Hasil
| Titik Ukur | Heap (MB) | Δ dari Baseline |
|---|---|---|
| Baseline (load awal) | [DATA CONTOH] 24.1 | – |
| Setelah 2 siklus | [DATA CONTOH] 25.3 | +1.2 |
| Setelah 4 siklus | [DATA CONTOH] 25.8 | +1.7 |
| Setelah 6 siklus | [DATA CONTOH] 26.0 | +1.9 |
| Setelah 8 siklus | [DATA CONTOH] 26.1 | +2.0 |
| Setelah 10 siklus | [DATA CONTOH] 26.2 | +2.1 |

### Cara Analisis Hasil
1. Plot heap size vs jumlah siklus — kurva yang **melandai (plateau)** setelah beberapa siklus = normal (caching/JIT warm-up), kurva **linear terus naik** = indikasi leak nyata.
2. Jika heap terus naik, periksa kandidat penyebab di kode: `simNodesRef` di `GraphCanvas.tsx` — meski map dibangun ulang tiap render (`next = new Map()`), node lama yang sudah tidak ada di `narrators` seharusnya otomatis tidak ter-copy; verifikasi ini benar dengan heap snapshot comparison (cari retainer path pada objek `SimNode` lama).
3. Bandingkan dengan baseline memori tab browser kosong untuk konteks skala (mis. total tab Chrome idle ~30-50MB, jadi tambahan beberapa MB untuk graph 50-node tergolong wajar).

### Contoh Penulisan Bab III
> Pengujian penggunaan memori dilakukan dengan mengukur ukuran *JavaScript heap* aktif menggunakan Chrome DevTools Protocol (`Performance.getMetrics`) melalui Playwright. *Garbage collection* dipaksa dijalankan sebelum setiap pengukuran untuk memastikan angka yang diperoleh merepresentasikan memori yang benar-benar masih direferensikan (bukan sampah sementara). Pengujian mensimulasikan interaksi berulang berupa pergantian metrik sentralitas sebanyak 10 siklus, dengan pengambilan sampel setiap 2 siklus, untuk mendeteksi kemungkinan kebocoran memori (*memory leak*) akibat proses *refetch* dan rekonstruksi struktur simulasi fisika pada setiap perubahan state.

### Contoh Penulisan Bab IV
> Hasil pengujian menunjukkan penggunaan heap meningkat dari **[DATA CONTOH] 24,1 MB** pada kondisi awal menjadi **26,2 MB** setelah 10 siklus pergantian metrik, atau kenaikan sebesar 2,1 MB. Pola kenaikan melandai setelah siklus ke-6 (Δ dari 1,9 MB → 2,1 MB pada 4 siklus terakhir) mengindikasikan bahwa kenaikan ini bersifat *warm-up* wajar dan bukan kebocoran memori progresif, karena tidak menunjukkan tren linear tak terbatas. Temuan ini konsisten dengan desain `simNodesRef` pada `GraphCanvas.tsx` yang membangun ulang peta node simulasi berdasarkan daftar `narrators` terkini pada setiap render, sehingga node yang tidak lagi relevan otomatis tidak tereferensikan.

---

## 3.4 Response Time API Supabase

### Tujuan
Mengukur latensi masing-masing fungsi query di `src/app/data/api.ts` secara terisolasi (di luar overhead render browser), untuk mengidentifikasi endpoint mana yang paling lambat.

### Alat yang Digunakan
- **Script Node.js** dengan `@supabase/supabase-js` (pengukuran presisi, konsisten, tanpa noise UI)
- **Browser DevTools → Network tab** (verifikasi waterfall dari sisi klien nyata)

### Langkah-Langkah Pengujian
1. Buat script Node yang memanggil query yang **identik** dengan yang dipakai tiap fungsi di `api.ts` (server-side, pakai kredensial `.env` yang sama).
2. Ukur 5 endpoint utama: `fetchTopNarrators`, `fetchRelationsForNarrators`, `fetchEdgeRelations`, `fetchPerawiTable`, `fetchAnalyticsData`.
3. Ulangi setiap panggilan **30 kali**, hitung min/mean/median/p95/p99/max.
4. Jalankan dari mesin dengan koneksi stabil, di luar jam sibuk, agar hasil tidak bias oleh jaringan lokal.

### Kode yang Diperlukan

```js
// scripts/analysis/response-time.mjs
import { createClient } from '@supabase/supabase-js';
import 'dotenv/config';

const supabase = createClient(
  process.env.VITE_SUPABASE_URL,
  process.env.VITE_SUPABASE_ANON_KEY
);

async function timeIt(label, fn, repeat = 30) {
  const durations = [];
  for (let i = 0; i < repeat; i++) {
    const start = performance.now();
    const { error } = await fn();
    if (error) throw new Error(`${label}: ${error.message}`);
    durations.push(performance.now() - start);
  }
  durations.sort((a, b) => a - b);
  const avg = durations.reduce((a, b) => a + b, 0) / durations.length;
  const pct = (p) => durations[Math.floor(durations.length * p)];
  console.log(
    `${label.padEnd(32)} min=${durations[0].toFixed(0)}ms  ` +
    `avg=${avg.toFixed(0)}ms  median=${pct(0.5).toFixed(0)}ms  ` +
    `p95=${pct(0.95).toFixed(0)}ms  p99=${pct(0.99).toFixed(0)}ms  ` +
    `max=${durations.at(-1).toFixed(0)}ms`
  );
  return durations;
}

const SELECT_COLS =
  'Perawi_ID, Nama_Perawi, Perawi, In_Degree_Centrality, Out_Degree_Centrality, Eigenvector_Centrality, Closeness_Centrality, Betweenness_Centrality';

await timeIt('fetchTopNarrators(inDegree,50)', () =>
  supabase.from('Sentralitas').select(SELECT_COLS)
    .order('In_Degree_Centrality', { ascending: false }).limit(50)
);

await timeIt('fetchRelationsForNarrators (query 1/2: Sentralitas)', () =>
  supabase.from('Sentralitas').select('Perawi_ID, Nama_Perawi').limit(50)
);

await timeIt('fetchEdgeRelations (page 0, size 1000)', () =>
  supabase.from('Edge').select('Edge_id, Guru_Id, Murid_Id, Weight', { count: 'exact' })
    .order('Weight', { ascending: false }).range(0, 999)
);

await timeIt('fetchPerawiTable (page 0, size 50)', () =>
  supabase.from('Sentralitas').select(SELECT_COLS, { count: 'exact' })
    .order('In_Degree_Centrality', { ascending: false }).range(0, 49)
);

await timeIt('fetchAnalyticsData (count Sentralitas)', () =>
  supabase.from('Sentralitas').select('*', { count: 'exact', head: true })
);
```

Jalankan: `node scripts/analysis/response-time.mjs`

### Parameter yang Diukur
| Parameter | Satuan |
|---|---|
| Response time min / mean / median / p95 / p99 / max | ms |
| Jumlah round-trip per operasi logis | request |

### Contoh Tabel Hasil
| Endpoint | Min | Mean | Median | P95 | P99 | Max |
|---|---|---|---|---|---|---|
| fetchTopNarrators (50) | [DATA CONTOH] 88 | 132 | 121 | 210 | 265 | 310 |
| fetchRelationsForNarrators (2 query) | [DATA CONTOH] 145 | 240 | 225 | 380 | 420 | 510 |
| fetchEdgeRelations (1000 baris) | [DATA CONTOH] 110 | 175 | 165 | 260 | 300 | 340 |
| fetchPerawiTable (50 baris) | [DATA CONTOH] 90 | 140 | 130 | 220 | 270 | 320 |
| fetchAnalyticsData (count) | [DATA CONTOH] 60 | 95 | 88 | 150 | 180 | 210 |

### Cara Analisis Hasil
1. Bandingkan `fetchRelationsForNarrators` (2 round-trip berurutan) terhadap endpoint lain — latensinya diprediksi ~2× lipat karena sifatnya sekuensial (`api.ts:80-104`), bukan paralel.
2. Terapkan ambang RAIL model Google (<100ms instan, <1000ms batas atas dapat diterima) untuk menilai kelayakan tiap endpoint.
3. Gunakan p95/p99 (bukan hanya mean) untuk menilai *worst-case* pengalaman pengguna — mean bisa menyembunyikan lonjakan sesekali.
4. Silangkan dengan §3.8 (Query Performance) — jika response time tinggi tapi query plan efisien, latensi kemungkinan didominasi jaringan/round-trip, bukan komputasi database.

### Contoh Penulisan Bab III
> Pengujian *response time* dilakukan secara terisolasi dari lapisan antarmuka dengan menjalankan panggilan query yang identik dengan implementasi pada `src/app/data/api.ts` melalui skrip Node.js menggunakan pustaka `@supabase/supabase-js`. Setiap endpoint diuji sebanyak 30 kali pengulangan untuk memperoleh distribusi latensi yang representatif, dengan metrik yang dilaporkan meliputi nilai minimum, rata-rata, median, persentil ke-95, persentil ke-99, dan maksimum.

### Contoh Penulisan Bab IV
> Pengujian menunjukkan bahwa fungsi `fetchRelationsForNarrators` memiliki latensi tertinggi dengan rata-rata **[DATA CONTOH] 240 ms** dan P95 sebesar 380 ms, hampir dua kali lipat dibanding `fetchTopNarrators` (rata-rata 132 ms). Hal ini sesuai dengan struktur implementasinya yang melakukan dua *round-trip* basis data secara berurutan — mengambil `Perawi_ID` dari tabel `Sentralitas`, kemudian menggunakan hasilnya untuk memfilter tabel `Edge`. Seluruh endpoint yang diuji tetap berada di bawah ambang 1.000 ms yang menurut model RAIL Google masih tergolong dapat diterima untuk operasi asinkron, meskipun `fetchRelationsForNarrators` menjadi kandidat utama untuk optimasi lanjutan, misalnya dengan menggabungkan kedua query menjadi satu melalui *PostgreSQL function* (RPC) di sisi Supabase.

---

## 3.5 Load Testing API

### Tujuan
Menguji ketahanan API Supabase terhadap beban pengguna bersamaan (*concurrent users*), mensimulasikan banyak pengguna membuka dashboard secara bersamaan, untuk menemukan titik jenuh (*breaking point*) dan memastikan sistem tetap responsif pada beban tinggi.

### Alat yang Digunakan
- **k6** (Grafana) — industri standar untuk load testing berbasis skrip JavaScript

### Langkah-Langkah Pengujian
1. Target endpoint: **PostgREST API Supabase langsung** (`https://<project>.supabase.co/rest/v1/...`) menggunakan header `apikey` (anon key) — merepresentasikan query yang sama seperti `fetchTopNarrators`.
2. Definisikan skenario *ramping virtual users* (VU): naik bertahap 0 → 10 → 50 → 100 VU, tiap tahap ditahan beberapa saat, lalu turun ke 0.
3. Tiap VU mensimulasikan pola akses nyata: request top-50 narrators, lalu request relasi (Edge), lalu jeda (`sleep`) seperti pengguna asli.
4. Tetapkan *threshold* keberhasilan: p95 latency < 1000ms, error rate < 1%.
5. Jalankan dan amati pada tahap VU berapa threshold mulai dilanggar.

### Kode yang Diperlukan

```js
// scripts/load-test/k6-load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  scenarios: {
    ramping_users: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '30s', target: 10 },
        { duration: '1m', target: 50 },
        { duration: '1m', target: 100 },
        { duration: '30s', target: 0 },
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<1000'],
    http_req_failed: ['rate<0.01'],
  },
};

const SUPABASE_URL = __ENV.SUPABASE_URL;
const ANON_KEY = __ENV.SUPABASE_ANON_KEY;

export default function () {
  const headers = { apikey: ANON_KEY, Authorization: `Bearer ${ANON_KEY}` };

  const cols = 'Perawi_ID,Nama_Perawi,Perawi,In_Degree_Centrality,Out_Degree_Centrality,Eigenvector_Centrality,Closeness_Centrality,Betweenness_Centrality';
  const res1 = http.get(
    `${SUPABASE_URL}/rest/v1/Sentralitas?select=${cols}&order=In_Degree_Centrality.desc&limit=50`,
    { headers, tags: { name: 'fetchTopNarrators' } }
  );
  check(res1, { 'status 200 (narrators)': (r) => r.status === 200 });

  const res2 = http.get(
    `${SUPABASE_URL}/rest/v1/Edge?select=Guru_Id,Murid_Id,Weight&limit=1000`,
    { headers, tags: { name: 'fetchEdges' } }
  );
  check(res2, { 'status 200 (edges)': (r) => r.status === 200 });

  sleep(1);
}
```

Jalankan (Windows PowerShell):
```powershell
$env:SUPABASE_URL="https://xxxx.supabase.co"
$env:SUPABASE_ANON_KEY="eyJ..."
k6 run scripts/load-test/k6-load-test.js
```

### Parameter yang Diukur
| Parameter | Satuan |
|---|---|
| Throughput (requests/sec) | req/s |
| Error rate | % |
| Latency p95 / p99 per tahap VU | ms |
| Breaking point (VU saat threshold dilanggar) | jumlah VU |

### Contoh Tabel Hasil
| Tahap (VU) | Throughput (req/s) | Error Rate | P95 Latency (ms) | Status |
|---|---|---|---|---|
| 10 | [DATA CONTOH] 9.8 | 0% | 320 | ✅ Lolos |
| 50 | [DATA CONTOH] 46.2 | 0.2% | 680 | ✅ Lolos |
| 100 | [DATA CONTOH] 88.5 | 3.1% | 1.450 | ❌ Gagal (p95 & error rate) |

### Cara Analisis Hasil
1. Identifikasi VU pertama kali threshold dilanggar → itulah kapasitas praktis sistem saat ini.
2. Periksa apakah error berasal dari rate-limiting Supabase (kode 429) atau timeout koneksi — beda akar masalah, beda solusi (upgrade tier vs optimasi query).
3. Bandingkan throughput vs jumlah VU — jika throughput berhenti naik meski VU terus ditambah (plateau), itu tanda saturasi di sisi server.
4. Hubungkan dengan §3.9 — load testing menguji *concurrent users* pada N=50 tetap; jika nanti N diperbesar, ulangi pengujian ini karena payload per-request akan lebih besar.

### Contoh Penulisan Bab III
> Pengujian beban (*load testing*) dilakukan menggunakan k6 untuk mensimulasikan permintaan bersamaan langsung ke REST API Supabase, merepresentasikan pola akses yang identik dengan yang dilakukan aplikasi saat memuat data 50 perawi teratas beserta relasinya. Skenario pengujian menggunakan strategi *ramping virtual users*, dengan jumlah pengguna virtual dinaikkan secara bertahap dari 0 hingga 100 dalam beberapa tahap, untuk mengamati perubahan throughput, tingkat kegagalan, dan latensi seiring peningkatan beban. Kriteria keberhasilan ditetapkan sebagai latensi persentil ke-95 di bawah 1.000 ms dan tingkat kegagalan di bawah 1%.

### Contoh Penulisan Bab IV
> Hasil pengujian beban menunjukkan sistem mampu menangani hingga **50 pengguna virtual bersamaan** dengan latensi P95 sebesar **[DATA CONTOH] 680 ms** dan tingkat kegagalan 0,2%, masih memenuhi kriteria yang ditetapkan. Namun pada beban **100 pengguna virtual**, latensi P95 meningkat menjadi **1.450 ms** dan tingkat kegagalan naik menjadi **3,1%**, melampaui ambang batas yang ditetapkan. Titik jenuh (*breaking point*) sistem berada di antara 50–100 pengguna bersamaan pada konfigurasi Supabase tingkat *free/pro* yang digunakan saat pengujian, mengindikasikan perlunya strategi *caching* atau *connection pooling* jika target pengguna simultan diperkirakan melebihi 50.

---

## 3.6 Graph Completeness

### Tujuan
Memverifikasi bahwa graph yang divisualisasikan **benar-benar merepresentasikan** data ground-truth di database — tidak ada node atau edge yang hilang secara tidak sengaja akibat bug (di luar pembatasan top-50 yang memang disengaja).

### Alat yang Digunakan
- **Script Node.js** — query langsung ke Supabase sebagai *ground truth*, dibandingkan dengan hasil fungsi aplikasi

### Langkah-Langkah Pengujian
1. Tentukan *ground truth* node: query manual `Sentralitas` ORDER BY metrik DESC LIMIT 50 → catat 50 `Perawi_ID`.
2. Bandingkan dengan output `fetchTopNarrators(metric, 50)` — jumlah dan identitas harus identik.
3. Tentukan *ground truth* edge: query manual `Edge` WHERE `Guru_Id` IN (50 id) AND `Murid_Id` IN (50 id) → hitung jumlah baris (setelah dedup).
4. Bandingkan dengan output `fetchRelationsForNarrators` — jumlah relasi harus identik (dan cek limit 5000 di `api.ts:102` tidak pernah terlampaui — jika iya, ada data yang terpotong diam-diam).
5. Verifikasi tidak ada node dengan koordinat `NaN`/`Infinity` yang membuatnya gagal digambar (`GraphCanvas.tsx:378` sudah ada guard `!isFinite`, tapi perlu dicek node yang di-skip ini tercatat sebagai insiden, bukan hilang tanpa jejak).

### Kode yang Diperlukan

```js
// scripts/analysis/completeness-check.mjs
import { createClient } from '@supabase/supabase-js';
import 'dotenv/config';

const supabase = createClient(process.env.VITE_SUPABASE_URL, process.env.VITE_SUPABASE_ANON_KEY);
const METRIC_COLUMN = 'In_Degree_Centrality';
const N = 50;

async function main() {
  // 1. Ground truth node
  const { data: groundTruthNodes, error: e1 } = await supabase
    .from('Sentralitas').select('Perawi_ID, Nama_Perawi')
    .order(METRIC_COLUMN, { ascending: false }).limit(N);
  if (e1) throw e1;
  const groundTruthIds = new Set(groundTruthNodes.map(r => r.Perawi_ID));

  // 2. Ground truth edge (kedua ujung dalam top-N)
  const idsArr = [...groundTruthIds];
  const { data: groundTruthEdges, error: e2 } = await supabase
    .from('Edge').select('Guru_Id, Murid_Id')
    .in('Guru_Id', idsArr).in('Murid_Id', idsArr);
  if (e2) throw e2;

  // 3. Hasil dari fungsi aplikasi (panggil endpoint yang sama seperti useGraphData)
  const { data: appNodes } = await supabase
    .from('Sentralitas').select('Perawi_ID, Nama_Perawi')
    .order(METRIC_COLUMN, { ascending: false }).limit(N);
  const appNames = appNodes.map(n => n.Nama_Perawi);
  const { data: perawiRes } = await supabase
    .from('Sentralitas').select('Perawi_ID, Nama_Perawi').in('Nama_Perawi', appNames);
  const appIds = perawiRes.map(p => p.Perawi_ID);
  const { data: appEdges } = await supabase
    .from('Edge').select('Guru_Id, Murid_Id').in('Guru_Id', appIds).in('Murid_Id', appIds).limit(5000);

  // 4. Bandingkan
  const nodeMatch = groundTruthIds.size === new Set(appIds).size &&
    [...groundTruthIds].every(id => appIds.includes(id));
  const nodeCompleteness = (new Set(appIds).size / groundTruthIds.size * 100).toFixed(1);
  const edgeCompleteness = (appEdges.length / groundTruthEdges.length * 100).toFixed(1);

  console.log(`Node ground-truth: ${groundTruthIds.size} | Node aplikasi: ${new Set(appIds).size} | Match: ${nodeMatch}`);
  console.log(`Edge ground-truth: ${groundTruthEdges.length} | Edge aplikasi: ${appEdges.length}`);
  console.log(`Node completeness: ${nodeCompleteness}% | Edge completeness: ${edgeCompleteness}%`);
  console.log(`Edge query limit (5000) terlampaui: ${appEdges.length >= 5000 ? 'YA — DATA TERPOTONG!' : 'Tidak'}`);
}
main();
```

### Parameter yang Diukur
| Parameter | Satuan |
|---|---|
| Node completeness ratio | % (aplikasi / ground truth) |
| Edge completeness ratio | % |
| Jumlah node/edge hilang | count |
| Insiden limit query terlampaui | boolean |

### Contoh Tabel Hasil
| Metrik Diuji | Node GT | Node App | Edge GT | Edge App | Node Completeness | Edge Completeness |
|---|---|---|---|---|---|---|
| In Degree Centrality | 50 | [DATA CONTOH] 50 | 612 | [DATA CONTOH] 612 | 100% | 100% |
| Betweenness Centrality | 50 | [DATA CONTOH] 50 | 578 | [DATA CONTOH] 578 | 100% | 100% |

### Cara Analisis Hasil
1. Completeness < 100% pada node = bug kritis (fungsi `fetchTopNarrators` tidak mengambil data yang seharusnya).
2. Completeness < 100% pada edge lebih sering disebabkan dedup yang salah atau limit query terlampaui — periksa log "Edge query limit terlampaui".
3. Completeness = 100% pada seluruh 5 metrik sentralitas mengonfirmasi bahwa pembatasan ke 50 node adalah **keputusan desain yang konsisten dieksekusi**, bukan kebocoran data akibat bug — poin penting untuk dijelaskan di Bab IV agar pembaca skripsi tidak salah paham antara *scoping* yang disengaja vs *incompleteness* yang merupakan cacat.

### Contoh Penulisan Bab III
> Pengujian *graph completeness* dilakukan dengan membandingkan data yang ditampilkan aplikasi terhadap *ground truth* yang diperoleh melalui query langsung ke basis data Supabase. Untuk setiap metrik sentralitas, 50 node dengan nilai tertinggi diambil sebagai acuan, kemudian jumlah dan identitas node maupun relasi (edge) yang dihasilkan fungsi `fetchTopNarrators` dan `fetchRelationsForNarrators` dibandingkan dengan acuan tersebut. Rasio kelengkapan dihitung sebagai persentase kecocokan antara data aplikasi dan data acuan.

### Contoh Penulisan Bab IV
> Pengujian kelengkapan graph pada seluruh lima metrik sentralitas menunjukkan tingkat kesesuaian **100%** antara node dan edge yang ditampilkan aplikasi dengan data acuan di basis data — tidak ditemukan node maupun relasi yang hilang secara tidak disengaja. Perlu ditekankan bahwa pembatasan tampilan pada 50 node teratas merupakan keputusan desain yang eksplisit (lihat `useGraphData.ts` baris 23), bukan indikasi ketidaklengkapan data; seluruh 50 node dan relasi di antaranya berhasil direpresentasikan secara akurat dan menyeluruh sesuai cakupan yang ditetapkan.

---

## 3.7 Graph Consistency

### Tujuan
Memastikan struktur graph konsisten secara internal (tidak ada edge yang merujuk node tak dikenal, tidak ada duplikasi tak disengaja) **dan** konsisten antar-eksekusi (hasil yang sama pada pemanggilan berulang) — poin krusial karena `fetchTopNarrators` melakukan `ORDER BY <kolom> DESC LIMIT 50` **tanpa secondary sort key** (`api.ts:58-62`), yang secara teoretis dapat menghasilkan urutan berbeda saat ada nilai kembar di batas peringkat ke-50.

### Alat yang Digunakan
- **Script Node.js** — referential integrity check + repeatability check

### Langkah-Langkah Pengujian
1. **Referential integrity**: untuk setiap `relation` yang dikembalikan `fetchRelationsForNarrators`, verifikasi `source` dan `target` keduanya ada di daftar `narrators` yang sama.
2. **Duplikasi**: verifikasi tidak ada pasangan `(source, target)` yang muncul lebih dari sekali (di luar agregasi weight yang memang disengaja).
3. **Determinisme**: panggil `fetchTopNarrators(metric, 50)` sebanyak 5 kali berturut-turut untuk setiap metrik, bandingkan set `Perawi_ID` yang dihasilkan — harus identik di semua run.
4. **Konsistensi arah**: ambil 5 relasi sampel, verifikasi manual terhadap sumber data sanad bahwa arah `source → target` (murid → guru, sesuai komentar `api.ts:118-119`) benar secara domain.

### Kode yang Diperlukan

```js
// scripts/analysis/consistency-check.mjs
import { createClient } from '@supabase/supabase-js';
import 'dotenv/config';

const supabase = createClient(process.env.VITE_SUPABASE_URL, process.env.VITE_SUPABASE_ANON_KEY);
const METRICS = ['In_Degree_Centrality', 'Out_Degree_Centrality', 'Eigenvector_Centrality', 'Closeness_Centrality', 'Betweenness_Centrality'];

async function checkDeterminism(metricCol, n = 50, runs = 5) {
  const results = [];
  for (let i = 0; i < runs; i++) {
    const { data } = await supabase
      .from('Sentralitas').select('Perawi_ID')
      .order(metricCol, { ascending: false }).limit(n);
    results.push(data.map(r => r.Perawi_ID).sort((a, b) => a - b).join(','));
  }
  const uniqueResults = new Set(results);
  return { metricCol, consistent: uniqueResults.size === 1, variants: uniqueResults.size, runs };
}

async function checkReferentialIntegrity(metricCol, n = 50) {
  const { data: nodes } = await supabase
    .from('Sentralitas').select('Perawi_ID, Nama_Perawi')
    .order(metricCol, { ascending: false }).limit(n);
  const ids = nodes.map(x => x.Perawi_ID);
  const idToName = new Map(nodes.map(x => [x.Perawi_ID, x.Nama_Perawi]));

  const { data: edges } = await supabase
    .from('Edge').select('Guru_Id, Murid_Id')
    .in('Guru_Id', ids).in('Murid_Id', ids).limit(5000);

  const dangling = edges.filter(e => !idToName.has(e.Guru_Id) || !idToName.has(e.Murid_Id));
  const seen = new Set();
  const duplicates = edges.filter(e => {
    const key = `${e.Guru_Id}|${e.Murid_Id}`;
    if (seen.has(key)) return true;
    seen.add(key);
    return false;
  });

  return { metricCol, totalEdges: edges.length, dangling: dangling.length, duplicates: duplicates.length };
}

for (const metric of METRICS) {
  const det = await checkDeterminism(metric);
  console.log(
    `[Determinisme] ${metric}: ${det.consistent ? 'KONSISTEN' : `TIDAK KONSISTEN (${det.variants} varian dari ${det.runs} run)`}`
  );
  const integ = await checkReferentialIntegrity(metric);
  console.log(
    `[Integritas]   ${metric}: ${integ.totalEdges} edge | dangling=${integ.dangling} | duplikat=${integ.duplicates}`
  );
}
```

### Parameter yang Diukur
| Parameter | Satuan |
|---|---|
| Dangling edge count | jumlah |
| Duplicate edge count | jumlah |
| Determinisme antar-run | % run identik |
| Konsistensi arah edge (spot-check manual) | pass/fail |

### Contoh Tabel Hasil
| Metrik | Dangling Edge | Duplicate Edge | Determinisme (5 run) |
|---|---|---|---|
| In Degree Centrality | [DATA CONTOH] 0 | 0 | ✅ Konsisten |
| Out Degree Centrality | [DATA CONTOH] 0 | 0 | ✅ Konsisten |
| Betweenness Centrality | [DATA CONTOH] 0 | 0 | ⚠️ Tidak konsisten (2 varian — ada nilai kembar di peringkat ke-50) |

### Cara Analisis Hasil
1. `Dangling edge > 0` = bug serius pada logika filter `fetchRelationsForNarrators` (seharusnya mustahil karena query sudah memfilter `IN ids` di kedua sisi — jika terjadi, curigai *stale closure* atau race condition antar-request).
2. `Duplicate edge > 0` = periksa logika dedup `api.ts:109-116` — kemungkinan `Guru_Id|Murid_Id` yang sama muncul di lebih dari satu baris mentah karena data sumber memang punya duplikat.
3. **Temuan determinisme tidak konsisten** adalah hal paling penting untuk dibahas: ini bug nyata pada `api.ts:58-62` yang tidak memakai *secondary sort key*. Rekomendasi perbaikan: tambahkan `.order('Perawi_ID', { ascending: true })` sebagai tie-breaker setelah `.order(orderBy, ...)`.

### Contoh Penulisan Bab III
> Pengujian konsistensi graph mencakup dua aspek: **integritas referensial**, yaitu memastikan setiap relasi (edge) yang ditampilkan merujuk pada node yang benar-benar ada dalam himpunan node yang sama tanpa duplikasi; dan **determinisme**, yaitu memastikan pemanggilan berulang terhadap fungsi pengambilan data top-N menghasilkan himpunan node yang identik. Determinisme diuji dengan memanggil fungsi pengambilan data sebanyak 5 kali untuk setiap metrik sentralitas, kemudian membandingkan kesamaan himpunan `Perawi_ID` yang dihasilkan pada setiap panggilan.

### Contoh Penulisan Bab IV
> Pengujian integritas referensial pada seluruh lima metrik sentralitas tidak menemukan adanya *dangling edge* maupun duplikasi relasi, menunjukkan bahwa logika penyaringan pada `fetchRelationsForNarrators` bekerja dengan benar. Namun, pengujian determinisme menemukan bahwa metrik **Betweenness Centrality** menghasilkan **2 varian himpunan node berbeda** dari 5 kali pengulangan pemanggilan yang identik. Temuan ini disebabkan oleh implementasi query pada `fetchTopNarrators` (`api.ts` baris 58–62) yang hanya menggunakan satu kolom pengurutan (`ORDER BY ... LIMIT 50`) tanpa kolom penentu urutan kedua (*secondary sort key*), sehingga PostgreSQL tidak menjamin urutan yang deterministik ketika terdapat nilai kembar tepat di batas peringkat ke-50. Sebagai rekomendasi perbaikan, query perlu ditambahkan `ORDER BY <metrik> DESC, Perawi_ID ASC` untuk menjamin hasil yang konsisten pada setiap pemuatan halaman.

---

## 3.8 Query Performance

### Tujuan
Menganalisis efisiensi eksekusi query SQL yang mendasari fungsi-fungsi di `api.ts` pada level basis data — rencana eksekusi (*query plan*), penggunaan indeks, dan waktu eksekusi murni di sisi PostgreSQL (terpisah dari latensi jaringan yang sudah dibahas di §3.4).

### Alat yang Digunakan
- **Supabase SQL Editor** (dashboard project)
- **PostgreSQL `EXPLAIN ANALYZE`**

### Langkah-Langkah Pengujian
1. Buka **Supabase Dashboard → SQL Editor**.
2. Jalankan `EXPLAIN ANALYZE` untuk setiap pola query utama yang dipakai `api.ts`.
3. Catat: tipe scan (`Seq Scan` vs `Index Scan`), *planning time*, *execution time*, jumlah baris di-scan vs dikembalikan.
4. Untuk query dengan `Seq Scan` pada tabel besar, buat indeks yang sesuai, lalu ulangi `EXPLAIN ANALYZE` untuk perbandingan before/after.

### Kode yang Diperlukan

```sql
-- scripts/sql/query-analysis.sql

-- Query A: fetchTopNarrators — order by kolom sentralitas + limit
EXPLAIN ANALYZE
SELECT "Perawi_ID","Nama_Perawi","Perawi",
       "In_Degree_Centrality","Out_Degree_Centrality",
       "Eigenvector_Centrality","Closeness_Centrality","Betweenness_Centrality"
FROM "Sentralitas"
ORDER BY "In_Degree_Centrality" DESC
LIMIT 50;

-- Query B: fetchRelationsForNarrators (tahap 2) — double IN filter pada Edge
EXPLAIN ANALYZE
SELECT "Guru_Id","Murid_Id","Weight"
FROM "Edge"
WHERE "Guru_Id" = ANY(ARRAY[1,2,3 /* ...50 id */])
  AND "Murid_Id" = ANY(ARRAY[1,2,3 /* ...50 id */])
LIMIT 5000;

-- Query C: searchPerawiByName — ILIKE dengan wildcard di awal
EXPLAIN ANALYZE
SELECT "Perawi_ID" FROM "Sentralitas"
WHERE "Nama_Perawi" ILIKE '%ibn%'
LIMIT 200;

-- Query D: fetchNarratorCounts — count(*) exact
EXPLAIN ANALYZE
SELECT COUNT(*) FROM "Edge" WHERE "Guru_Id" = 123;

-- === Rekomendasi indeks (jalankan lalu ulangi query di atas) ===
CREATE INDEX IF NOT EXISTS idx_sentralitas_in_degree  ON "Sentralitas" ("In_Degree_Centrality" DESC);
CREATE INDEX IF NOT EXISTS idx_sentralitas_out_degree ON "Sentralitas" ("Out_Degree_Centrality" DESC);
CREATE INDEX IF NOT EXISTS idx_sentralitas_eigen      ON "Sentralitas" ("Eigenvector_Centrality" DESC);
CREATE INDEX IF NOT EXISTS idx_sentralitas_closeness  ON "Sentralitas" ("Closeness_Centrality" DESC);
CREATE INDEX IF NOT EXISTS idx_sentralitas_between    ON "Sentralitas" ("Betweenness_Centrality" DESC);

CREATE INDEX IF NOT EXISTS idx_edge_guru  ON "Edge" ("Guru_Id");
CREATE INDEX IF NOT EXISTS idx_edge_murid ON "Edge" ("Murid_Id");

-- ILIKE dengan wildcard awal butuh trigram index, bukan B-tree biasa
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX IF NOT EXISTS idx_sentralitas_nama_trgm
  ON "Sentralitas" USING gin ("Nama_Perawi" gin_trgm_ops);
```

### Parameter yang Diukur
| Parameter | Satuan |
|---|---|
| Execution time (dari `EXPLAIN ANALYZE`) | ms |
| Planning time | ms |
| Tipe scan (Seq Scan / Index Scan / Bitmap Scan) | kategori |
| Rows scanned vs rows returned (selektivitas) | rasio |

### Contoh Tabel Hasil — REAL, before/after lengkap (detail plan penuh di §A.8)
| Query | Sebelum Index (ms, tipe scan) | Sesudah Index (ms, tipe scan) | Peningkatan |
|---|---|---|---|
| A: Top-50 by In_Degree | 76,437 ms, Parallel Seq Scan + top-N heapsort | **1,435 ms**, Index Scan | **~53×** |
| B: Edge double-IN | 1.228,484 ms, Parallel Seq Scan (±389rb baris) | **35,639 ms**, BitmapAnd (2 index) | **~34×** |
| C: ILIKE search | 6,603 ms, Seq Scan (early-exit `LIMIT`) | **6,751 ms**, Bitmap Index Scan (GIN trgm) | **~tidak berubah** |
| D: Count by Guru_Id | 846,632 ms, Parallel Seq Scan (±389rb baris) | **2,322 ms**, Index Only Scan | **~365×** |

### Cara Analisis Hasil
1. `Seq Scan` pada tabel dengan ratusan ribu baris + `ORDER BY`/`WHERE ... IN` = kandidat kuat untuk indeks pada kolom yang bersangkutan — terbukti pada Query A, B, D yang mencatat percepatan 34×-365× setelah index ditambahkan.
2. Bandingkan *rows removed by filter* (tersedia di output `EXPLAIN ANALYZE`) — angka besar menunjukkan query tidak selektif tanpa indeks (Query B & D masing-masing memindai ±387 ribu baris untuk hasil yang jauh lebih kecil).
3. **Jangan asumsikan index selalu menguntungkan** — Query C membuktikan bahwa jika query sudah kebetulan diuntungkan mekanisme lain (di sini, *early-exit* `LIMIT` tanpa `ORDER BY`), index baru tidak memberi manfaat berarti untuk kasus spesifik yang diuji; evaluasi tetap perlu dilakukan per-query, bukan digeneralisasi.
4. Untuk dataset skripsi yang relatif statis (data sanad tidak sering berubah), trade-off *write overhead* dari indeks dapat diabaikan — indeks murni menguntungkan untuk mayoritas kasus di atas.
5. Jadikan tabel before/after ini sebagai bukti kuantitatif rekomendasi optimasi di Bab V.

### Contoh Penulisan Bab III
> Pengujian efisiensi query dilakukan menggunakan perintah `EXPLAIN ANALYZE` pada PostgreSQL melalui SQL Editor Supabase, untuk menganalisis rencana eksekusi (*query plan*), jenis pemindaian tabel yang digunakan (*sequential scan* atau *index scan*), serta waktu eksekusi aktual dari setiap pola query utama yang digunakan aplikasi. Pengujian dilakukan dalam dua kondisi: sebelum dan sesudah penambahan indeks pada kolom-kolom yang sering digunakan sebagai kriteria pengurutan (`ORDER BY`) dan penyaringan (`WHERE`/`IN`), untuk mengukur dampak kuantitatif optimasi indeks terhadap performa query.

### Contoh Penulisan Bab IV
> Analisis rencana eksekusi menunjukkan bahwa seluruh query utama aplikasi awalnya menggunakan **Sequential Scan** (dengan bantuan *parallel worker* pada beberapa kasus) karena belum terdapat indeks pada kolom-kolom yang menjadi kriteria pengurutan maupun penyaringan. Query pengambilan 50 node teratas berdasarkan `In_Degree_Centrality` membutuhkan waktu eksekusi 76,4 ms dengan memindai seluruh 104.145 baris tabel `Sentralitas`; setelah penambahan indeks B-tree, rencana eksekusi berubah menjadi **Index Scan** dengan waktu eksekusi turun menjadi **1,4 ms**, atau peningkatan sekitar **53 kali lipat**. Dampak serupa namun lebih besar ditemukan pada query pengambilan relasi (`Edge`), yang semula membutuhkan 1.228,5 ms akibat filter ganda pada kolom `Guru_Id` dan `Murid_Id` yang belum diindeks, memaksa pemindaian menyeluruh terhadap ±389.000 baris; setelah indeks ditambahkan pada kedua kolom, PostgreSQL memilih strategi **BitmapAnd** yang menggabungkan dua *bitmap index scan*, menurunkan waktu eksekusi menjadi 35,6 ms (**peningkatan ~34 kali lipat**). Perbaikan paling signifikan terjadi pada query penghitungan (`COUNT`) jumlah murid seorang perawi: dari 846,6 ms dengan *Sequential Scan* menjadi hanya **2,3 ms** dengan **Index Only Scan**, yaitu peningkatan performa sekitar **365 kali lipat**, karena PostgreSQL dapat menjawab query langsung dari struktur indeks tanpa perlu mengakses tabel heap. Menariknya, query pencarian nama menggunakan operator `ILIKE` **tidak menunjukkan perbaikan berarti** setelah penambahan indeks GIN berbasis ekstensi `pg_trgm` (6,6 ms menjadi 6,8 ms), karena performa awalnya sudah tergolong cepat akibat mekanisme *early-exit* pada klausa `LIMIT` tanpa `ORDER BY`, bukan karena efisiensi pemindaian tabel; temuan ini menegaskan bahwa manfaat indeks bersifat kondisional dan perlu dievaluasi per jenis query, bukan diasumsikan berlaku universal. Yang paling signifikan dari sisi validitas data, pengujian pembuktian (Query E) menemukan bahwa nilai kolom `Nama_Perawi` **tidak unik** — setidaknya 15 nama perawi berbeda memiliki lebih dari satu `Perawi_ID` pada tabel `Sentralitas`. Temuan ini mengonfirmasi bahwa strategi pencocokan data berbasis nama pada `fetchRelationsForNarrators` bukan risiko teoretis, melainkan **cacat arsitektural sistemik** yang berpotensi memengaruhi validitas graph pada setiap perhitungan ulang top-N narasumber.

---

## 3.9 Scalability Testing

### Tujuan
Mengukur bagaimana waktu load, FPS, penggunaan memori, dan waktu query berubah seiring pertambahan jumlah node (N), untuk menentukan batas praktis skalabilitas arsitektur saat ini — terutama karena loop repulsion `GraphCanvas.tsx:253-266` bersifat **O(N²)** dan jumlah edge berpotensi tumbuh kuadratik terhadap N.

### Alat yang Digunakan
- Kombinasi Playwright (§3.1–3.3) + skrip Node (§3.4) dijalankan berulang pada berbagai nilai N

### Prasyarat: Parameterisasi N (perubahan kode minimal, opsional)
Saat ini N di-hardcode di `useGraphData.ts:23`. Untuk pengujian skalabilitas, nilai ini perlu bisa diubah dari luar, contoh perubahan minimal (**tidak diterapkan otomatis — terapkan manual jika ingin menjalankan pengujian ini**):

```diff
 export function useGraphData(selectedMetric: string): GraphData {
+  const n = Number(new URLSearchParams(window.location.search).get('n')) || 50;
   const [narrators, setNarrators] = useState<Narrator[]>([]);
   ...
-    fetchTopNarrators(selectedMetric, 50)
+    fetchTopNarrators(selectedMetric, n)
       .then(async top50 => {
```

Dengan perubahan ini, N bisa diatur lewat URL: `http://localhost:5173/?n=200`.

### Langkah-Langkah Pengujian
1. Terapkan parameterisasi N di atas (di *branch*/lingkungan pengujian terpisah, bukan production).
2. Jalankan pengujian §3.1 (load time), §3.2 (FPS), §3.3 (memory) untuk **N = 50, 100, 250, 500, 1000**.
3. Untuk tiap N, catat juga jumlah edge aktual yang dihasilkan (untuk memverifikasi pertumbuhan kuadratik terhadap N).
4. Plot seluruh metrik terhadap N, cari titik infleksi (*elbow point*) tempat performa mulai menurun tajam — khususnya N di mana FPS jatuh di bawah 30.

### Kode yang Diperlukan

```ts
// tests/performance/scalability.spec.ts
import { test } from '@playwright/test';

const N_VALUES = [50, 100, 250, 500, 1000];

for (const n of N_VALUES) {
  test(`Skalabilitas N=${n}`, async ({ page }) => {
    const t0 = performance.now();
    await page.goto(`http://localhost:5173/?n=${n}`);
    await page.waitForSelector('text=Memuat data dari Database', { state: 'detached', timeout: 60000 });
    const loadTime = performance.now() - t0;

    await page.waitForTimeout(3500);

    // FPS idle 5 detik
    const fpsSamples: number[] = await page.evaluate(() => {
      return new Promise<number[]>((resolve) => {
        const samples: number[] = [];
        let frames = 0, lastSecond = performance.now();
        const start = performance.now();
        function tick() {
          frames++;
          const now = performance.now();
          if (now - lastSecond >= 1000) { samples.push(frames); frames = 0; lastSecond = now; }
          if (now - start < 5000) requestAnimationFrame(tick); else resolve(samples);
        }
        requestAnimationFrame(tick);
      });
    });
    const avgFps = fpsSamples.reduce((a, b) => a + b, 0) / fpsSamples.length;

    const client = await page.context().newCDPSession(page);
    const { metrics } = await client.send('Performance.getMetrics');
    const heapMB = metrics.find(m => m.name === 'JSHeapUsedSize')!.value / 1024 / 1024;

    console.log(`N=${n}: loadTime=${loadTime.toFixed(0)}ms avgFPS=${avgFps.toFixed(1)} heap=${heapMB.toFixed(1)}MB`);
  });
}
```

### Parameter yang Diukur
| Parameter | Satuan |
|---|---|
| Load time vs N | ms |
| FPS rata-rata vs N | fps |
| Heap memory vs N | MB |
| Jumlah edge vs N (verifikasi tren kuadratik) | count |
| Waktu 1 iterasi physics tick vs N | ms |

### Contoh Tabel Hasil — REAL (lihat §A.9 untuk narasi lengkap)
| N (diminta) | Node Diterima | Edge Diterima | Load Time (ms) | FPS Rata-rata | Heap (MB) |
|---|---|---|---|---|---|
| 50 | 50 | 1.000 (capped, Bug #2) | 9.407 *(cold start)* | 59,8 | 8,8 |
| 100 | 100 | 1.000 (capped) | 2.707 | 61,0 | 11,1 |
| 250 | 250 | 1.000 (capped) | 2.788 | 59,3 | 10,1 |
| 500 | 500 | 1.000 (capped) | 3.263 | 56,8 | 10,3 |
| 1000 | **0 — GAGAL TOTAL** | — | timeout (>45.000) | — | — |

### Cara Analisis Hasil
1. **Data nyata mengoreksi hipotesis awal**: FPS tidak menurun signifikan dari N=50 ke N=500 (tetap 57-61) — beban O(N²) pada `runTick()` ternyata bukan bottleneck utama di rentang ini. Jangan asumsikan degradasi kuadratik tanpa mengukur; pada kasus ini justru **layer fetch data** yang menjadi titik gagal, bukan rendering.
2. Untuk N=1000, klasifikasikan kegagalan sebagai **hard failure** (biner), bukan *soft degradation* — penting dibedakan di Bab IV karena implikasinya berbeda: degradasi bertahap bisa "ditoleransi sampai batas tertentu", kegagalan keras berarti ada ambang yang benar-benar tidak boleh dilewati tanpa perbaikan arsitektur.
3. Cari ambang pasti dengan menambah panjang query-string filter `.in()` secara bertahap (lihat §A.9: sukses di 700 nama/~13,6rb karakter, gagal di 800 nama/~15,5rb karakter) — ambang bergantung pada total karakter ter-*encode*, bukan N secara langsung, karena panjang nama perawi bervariasi.
4. Gunakan temuan ini sebagai **dasar kuantitatif rekomendasi Bab V**: migrasi dari client-side `.in()` filter besar menuju Postgres RPC/function yang melakukan JOIN di database — ini sekaligus memperbaiki Bug #1 (name collision), Bug #2 (Max Rows truncation), dan ambang kegagalan N di atas.

### Contoh Penulisan Bab III
> Pengujian skalabilitas dilakukan dengan memvariasikan jumlah node (N) yang dimuat sistem — 50, 100, 250, 500, dan 1000 — melalui parameterisasi pada `useGraphData.ts` (parameter `n` opsional via URL query string, default tetap 50). Pada setiap nilai N, diukur waktu load, *frame rate*, dan penggunaan memori menggunakan instrumentasi yang sama dengan §3.1-§3.3. Untuk mengisolasi apakah keterlambatan atau kegagalan terjadi di lapisan pengambilan data atau lapisan render, pengujian tambahan dilakukan langsung terhadap Supabase REST API di luar konteks peramban, dengan memvariasikan jumlah nilai pada klausa filter `IN` secara bertahap untuk menemukan ambang batas kegagalan secara presisi.

### Contoh Penulisan Bab IV
> Hasil pengujian skalabilitas tidak menunjukkan pola degradasi kuadratik yang diperkirakan sebelumnya berdasarkan kompleksitas algoritmik simulasi repulsion `O(N²)` pada `GraphCanvas.tsx`. *Frame rate* rata-rata tetap stabil pada kisaran **56,8-61,0 FPS** untuk N=50 hingga N=500, mengindikasikan bahwa lapisan rendering bukan merupakan titik lemah utama pada rentang tersebut. Sebaliknya, pengujian pada N=1000 mengalami **kegagalan total** — permintaan data tidak pernah selesai dalam batas waktu 45 detik. Investigasi lanjutan yang dilakukan langsung terhadap Supabase REST API (di luar konteks peramban) mengonfirmasi bahwa kegagalan disebabkan oleh fungsi `fetchRelationsForNarrators`, yang membangun klausa filter `IN` berbasis nama perawi dan meng-*encode*-nya ke dalam *query string* URL. Panjang *string* tersebut tumbuh proporsional terhadap N, mencapai **19.408 karakter** pada N=1000. Pengujian bertahap menunjukkan permintaan berhasil hingga 700 nilai (~13.600 karakter) namun gagal total (`fetch failed`) mulai 800 nilai (~15.500 karakter), mengindikasikan pelanggaran batas ukuran URL/*header* pada infrastruktur HTTP yang dilalui. Temuan ini mengoreksi hipotesis awal: bottleneck skalabilitas sistem bukan terletak pada kompleksitas algoritma rendering, melainkan pada arsitektur pengambilan data yang bergantung pada klausa filter berukuran tidak terbatas yang di-*encode* ke URL. Berdasarkan hal ini, sistem direkomendasikan **aman dioperasikan hingga sekitar N=500-600**, dengan perbaikan arsitektural berupa migrasi ke fungsi basis data (Postgres RPC) yang melakukan penggabungan (*join*) langsung di sisi server sebagai syarat untuk mendukung skala di atas ambang tersebut.

---

## 4. Ringkasan & Checklist Eksekusi

| # | Pengujian | Alat Utama | File yang Dibuat |
|---|---|---|---|
| 1 | Waktu Load Graph | Playwright | `tests/performance/load-time.spec.ts` |
| 2 | FPS | Playwright | `tests/performance/fps.spec.ts` |
| 3 | Memori Browser | Playwright + CDP | `tests/performance/memory.spec.ts` |
| 4 | Response Time API | Node + Supabase JS | `scripts/analysis/response-time.mjs` |
| 5 | Load Testing API | k6 | `scripts/load-test/k6-load-test.js` |
| 6 | Graph Completeness | Node + Supabase JS | `scripts/analysis/completeness-check.mjs` |
| 7 | Graph Consistency | Node + Supabase JS | `scripts/analysis/consistency-check.mjs` |
| 8 | Query Performance | Supabase SQL Editor | `scripts/sql/query-analysis.sql` |
| 9 | Scalability Testing | Playwright (gabungan §1–3) | `tests/performance/scalability.spec.ts` |

Semua kode di atas bersifat **eksternal terhadap aplikasi** (tidak mengubah `src/`), kecuali §9 yang butuh satu perubahan kecil opsional pada `useGraphData.ts` untuk memparameterisasi N — dan itu pun hanya perlu aktif selama sesi pengujian skalabilitas.
