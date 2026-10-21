# UTS Graf Sains Data: Sistem Rekomendasi Graf dengan Python Driver dan Neo4j

Repositori ini berisi dokumentasi lengkap, kode program, visualisasi graf, dan analisis empiris untuk **Ujian Tengah Semester (UTS) Mata Kuliah Graf Sains Data (3 SKS)** pada Program Studi S1 Sains Data, Fakultas Matematika dan Ilmu Pengetahuan Alam, Universitas Negeri Surabaya.

---

## Struktur Repositori

```text
├── 01_Tugas_Pendahuluan.ipynb          # Notebook Persiapan Env, Driver Neo4j, & Checkpoint
├── 02_UTS_Wrangling_EDA_Analytics.ipynb# Notebook Utama (Wrangling, VSM, Similarity, KNN, Evaluasi)
├── requirements.txt                    # Daftar dependensi resmi sesuai naskah soal
├── .gitignore                          # Menjaga keamanan file kredensial dan dataset besar
├── plot_user_projection.png            # Visualisasi graf proyeksi User-User (NetworkX)
├── plot_item_projection.png            # Visualisasi graf proyeksi Item-Item (NetworkX)
├── Plot sebelum similarity.png         # Visualisasi graf soal 2.1 no 8 (Interaksi Tak Langsung via Visitor)
├── Plot setelah similarity.png         # Visualisasi graf soal 2.1 no 8 (Relasi Langsung SIMILAR Antar Item)
└── data_filtered/
    ├── events_p1_train.csv             # Dataset hasil filter periode Training (P1)
    ├── events_p2_test.csv              # Dataset hasil filter periode Testing (P2)
    ├── user_projection.csv             # Hasil proyeksi graf User-User (bobot shared items)
    ├── item_projection.csv             # Hasil proyeksi graf Item-Item (bobot shared visitors)
    └── similarity_results.csv          # Hasil Cosine Similarity lengkap dengan daftar shared visitors
```

---

## Konfigurasi Periode Waktu Terpilih (Opsi No. 9)

Dataset yang digunakan bersumber dari **RetailRocket E-Commerce Dataset (Kaggle)** oleh Roman Zykov. Sesuai ketentuan kombinasi unik bebas bentrok:
- **Fold Training (P1)**: `18 Mei 2015` s/d `31 Mei 2015` (14 hari interaksi)
- **Fold Testing (P2)**: `01 Juni 2015` s/d `15 Juni 2015` (15 hari interaksi)

### Ringkasan Statistik Data Interaksi:
| Metrik | Fold Training (P1) | Fold Testing (P2) |
| :--- | :---: | :---: |
| **Total Baris Interaksi** | **294.333 baris** | **300.563 baris** |
| **Pengunjung Unik (Visitor)** | 159.344 | 160.017 |
| **Item Unik** | 78.812 | 79.896 |
| **Interaksi `view`** | 285.187 | 291.122 |
| **Interaksi `addtocart`** | 6.939 | 7.110 |
| **Interaksi `transaction`** | 2.207 | 2.331 |
| **Overlap Visitor (P1 & P2)** | \multicolumn{2}{c|}{**7.357 Pengunjung Aktif**} |

---

## Arsitektur Graf & Komposisi Subgraph (Neo4j: `183utsgraf`)

Data interaksi P1 diimpor ke dalam database Neo4j lokal menggunakan skema graf berarah (*directed bipartite graph*):
- **Simpul (Nodes)**:
  - `(:Visitor {visitorId})`: Simpul pengunjung e-commerce.
  - `(:Item {itemId})`: Simpul produk e-commerce.
- **Relasi Interaksi**:
  - `(v)-[:VIEWED {timestamp}]->(i)`
  - `(v)-[:ADDED_TO_CART {timestamp}]->(i)`
  - `(v)-[:BOUGHT {timestamp, transactionId}]->(i)`

### Komposisi Subgraph yang Terbentuk di Neo4j:
- **Komposisi Simpul**:
  - `Visitor`: **159.344 simpul**
  - `Item`: **78.812 simpul**
- **Komposisi Relasi**:
  - `VIEWED`: **285.187 relasi**
  - `ADDED_TO_CART`: **6.939 relasi**
  - `BOUGHT`: **2.207 relasi**
  - *Total Relasi Interaksi Terverifikasi*: **294.333 relasi**

---

## Vector Space Modelling (VSM) & Proyeksi Graf (NetworkX)

Pembobotan interaksi implisit: `view = 1`, `addtocart = 2`, `transaction = 3`.

| Karakteristik | VSM Simpul User (Visitor) | VSM Simpul Item |
| :--- | :---: | :---: |
| **Dimensi Ruang Vektor** | $78.812$ (Fitur Item) | $159.344$ (Fitur User) |
| **Bentuk Matriks (Shape)** | $(159344, 78812)$ | $(78812, 159344)$ |
| **Non-Zero Elements** | $231.919$ | $231.919$ |
| **Tingkat Sparsitas** | **$99.99815\%$** | **$99.99815\%$** |

### Visualisasi Graf Proyeksi Bipartit:
1. **Proyeksi User-User**: Menghubungkan pengunjung yang memiliki kesamaan dalam melihat produk.
2. **Proyeksi Item-Item**: Menghubungkan produk-produk yang sering dilihat/dibeli secara bersamaan (*co-occurrence*).

> **Kesimpulan Komparasi (Soal 2.1 No. 6)**:  
> **Proyeksi Item jauh lebih efektif dan efisien** dibandingkan User karena katalog produk ($78.812$) hanya separuh dari populasi pengunjung ($159.344$), sehingga perhitungan matriks pasangan $N^2$ **4 kali lebih hemat memori dan komputasi**, serta memiliki asosiasi produk yang jauh lebih stabil terhadap *cold-start user*.

---

## Perhitungan Similarity & Write-Back ke Neo4j

Cosine Similarity dihitung pada ruang proyeksi item, dan pasangan item dengan kemiripan signifikan disimpan kembali ke Neo4j sebagai edge baru berlabel `[:SIMILAR {score}]`.

### Hasil Query Top-5 Similarity dari Record Acuan Item `257251`:
| Acuan Item | Item Serupa | Similarity Score |
| :---: | :---: | :---: |
| `257251` | **`309652`** | **0.3746** |
| `257251` | **`191581`** | **0.3121** |
| `257251` | **`342530`** | **0.3112** |
| `257251` | **`407658`** | **0.2659** |
| `257251` | **`182785`** | **0.2580** |

---

## Analytics: Rekomendasi K-Nearest Neighbors ($k=5$) & Evaluasi

### 1. Pemilihan User Acak Ter-reproduksi (Seed: `64`)
- **User Target Terpilih**: **Visitor ID `1156908`** 

### 2. Rekomendasi Top-3 Items yang Diiklankan:
Menggunakan algoritma *Collaborative Graph KNN ($k=5$)*:
1. **Item `422425`** (Skor Dukungan Tetangga: 15 pengunjung serupa)
2. **Item `315545`** (Skor Dukungan Tetangga: 11 pengunjung serupa)
3. **Item `193828`** (Skor Dukungan Tetangga: 8 pengunjung serupa)

### 3. Hasil Evaluasi Metrik pada Fold Testing (P2):
- **Hit@5**: `1.0`
- **NDCG@5**: `0.6509`

---