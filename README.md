# Deteksi dan Penghitungan Pohon Sawit Otomatis dari Citra Orthomosaic Drone

![Python](https://img.shields.io/badge/Python-3.8%2B-blue)
![Platform](https://img.shields.io/badge/Platform-Google%20Colab-orange)
![License](https://img.shields.io/badge/License-MIT-green)
![Status](https://img.shields.io/badge/Status-Prototype-yellow)

## Deskripsi Proyek

Repositori ini memuat notebook untuk **deteksi dan penghitungan pohon sawit secara otomatis** dari citra *orthomosaic* hasil pemotretan drone, menggunakan model deteksi objek dari **Roboflow** (*Palm Oil Detection and Counting*, oleh Prasadnr, `model_id: palm-oil-detection-and-counting/5`).

Notebook ini merupakan **versi Google Colab** dari alur kerja aslinya yang berbasis **ArcGIS Pro/`arcpy`**. Karena `arcpy` bersifat proprietary dan hanya tersedia pada lingkungan ArcGIS Pro, seluruh fungsi baca-raster diganti menggunakan **rasterio** (open source, berbasis GDAL yang sama), sementara output diganti dari File Geodatabase (`.gdb`) menjadi **GeoPackage (`.gpkg`)** - satu file yang dapat langsung dibuka di ArcGIS Pro maupun QGIS tanpa konversi tambahan.

Dibandingkan versi sebelumnya, notebook ini menambahkan keluaran **bounding box** (bukan hanya titik pusat) serta **peta interaktif (HTML)** untuk pratinjau cepat hasil deteksi.

## Fitur Utama

1. Menghitung jumlah pohon sawit secara otomatis dari citra orthomosaic beresolusi tinggi.
2. Mendeteksi posisi tiap pohon dalam bentuk titik pusat maupun bounding box.
3. Menghapus deteksi ganda (*duplicate detection*) pada area tumpang tindih (*overlap*) antar tile citra.
4. Mengekspor hasil deteksi ke format GeoPackage (dua layer: titik & bounding box) yang kompatibel dengan ArcGIS Pro dan QGIS.
5. Menghasilkan peta interaktif (HTML mandiri) untuk pratinjau hasil deteksi tanpa perlu software GIS.

## Metodologi & Alur Kerja

Notebook dijalankan sepenuhnya di **Google Colab**, dengan alur kerja sebagai berikut:

1. **Langkah 0 - Instalasi & Mount Google Drive**
   * Instalasi pustaka: `rasterio`, `geopandas`, `shapely`, `folium`, `rio-cogeo`, `localtileserver`.
   * Mount Google Drive untuk membaca citra orthomosaic dan menyimpan hasil.
2. **Langkah 1 - Konfigurasi Parameter**
   * Pengaturan path citra input, path output, ukuran tile, overlap antar tile, ambang confidence, dan jarak deduplikasi (lihat tabel [Parameter Konfigurasi](#parameter-konfigurasi)).
3. **Langkah 2 - Fungsi Inferensi per Tile**
   * Mengirim satu tile citra (JPEG ter-*encode* base64) ke Roboflow Hosted API dan mengembalikan daftar prediksi (posisi, ukuran, confidence, kelas).
4. **Langkah 3 - Tiling Raster & Deteksi**
   * Membaca raster orthomosaic per-blok menggunakan `rasterio.windows.Window` (raster besar tetap aman diproses tanpa dimuat sekaligus ke memori).
   * Memotong raster menjadi tile berukuran `TILE_SIZE_PX` dengan overlap `OVERLAP_PX`, lalu mengirim tiap tile ke Roboflow.
   * Mengonversi titik pusat dan 4 sudut bounding box tiap deteksi dari koordinat piksel tile ke koordinat peta, mengikuti transform tile masing-masing.
5. **Langkah 4 - Penghapusan Deteksi Ganda**
   * Menyusun seluruh deteksi berdasarkan confidence tertinggi, lalu menghapus titik yang berjarak lebih dekat dari `DEDUP_DIST_M` menggunakan `scipy.spatial.cKDTree`.
   * Konversi jarak ke satuan meter menyesuaikan CRS raster secara otomatis (geografis maupun terproyeksi).
6. **Langkah 5 - Ekspor ke GeoPackage**
   * Menyimpan hasil akhir sebagai dua layer dalam satu file GeoPackage: titik pusat (`deteksi_pohon_sawit`) dan bounding box (`deteksi_pohon_sawit_bbox`).
7. **Langkah 6 - Peta Interaktif**
   * Menyusun peta HTML mandiri (menggunakan `folium`) dengan latar citra raster yang di-*downsample* untuk pratinjau ringan, dioverlay dengan titik dan bounding box hasil deteksi.

## Struktur Direktori Data (Google Drive)

```text
/My Drive/Sawit/
├── Citra_2.tif                          # citra orthomosaic input
├── Hasil_Sawit_roboflow.gpkg             # output GeoPackage (dibuat otomatis)
└── peta_interaktif_sawit_roboflow.html   # output peta interaktif (dibuat otomatis)
```

> Catatan: path pada notebook (`ORTHO_RASTER`, `OUTPUT_PATH`) mengikuti struktur folder `Sawit/` di atas. Sesuaikan variabel tersebut di bagian Konfigurasi jika struktur Drive Anda berbeda.

## Prasyarat Instalasi

Notebook membutuhkan pustaka Python berikut (diinstal otomatis pada Langkah 0):

```bash
pip install rasterio geopandas shapely folium rio-cogeo localtileserver
```

Notebook dijalankan di Google Colab dan memanfaatkan modul `google.colab.drive` untuk mount Google Drive, serta membutuhkan API key dari [Roboflow](https://roboflow.com).

## Parameter Konfigurasi

| Parameter | Keterangan |
|---|---|
| `ORTHO_RASTER` | Path citra orthomosaic input (`.tif`) di Google Drive |
| `OUTPUT_PATH` | Path output GeoPackage, dibuat otomatis |
| `TILE_SIZE_PX` | Ukuran tile (px), disamakan dengan ukuran input model (default 640) |
| `OVERLAP_PX` | Overlap antar tile (px), agar pohon di tepi tile tidak hilang/terpotong |
| `CONF_THRESHOLD` | Ambang confidence deteksi (0-100); naikkan jika banyak *false positive*, turunkan jika banyak pohon terlewat |
| `DEDUP_DIST_M` | Jarak minimal (meter) antar dua titik deteksi agar tidak dianggap 1 pohon yang sama; sesuaikan dengan jarak tanam sawit riil di lapangan |
| `ROBOFLOW_API_KEY` | API key Roboflow - **jangan ditanam langsung di notebook publik**, gunakan Colab Secrets |
| `ROBOFLOW_MODEL_ID` | ID model Roboflow yang digunakan (`palm-oil-detection-and-counting/5`) |

## Hasil / Output

Notebook menghasilkan:

1. **GeoPackage** (`Hasil_Sawit_roboflow.gpkg`) berisi dua layer: titik pusat pohon dan bounding box pohon, lengkap dengan atribut `id_pohon`, `confidence`, dan `kelas`.
2. **Peta interaktif** (`peta_interaktif_sawit_roboflow.html`) - file mandiri yang dapat dibuka di browser mana pun tanpa software GIS, menampilkan pratinjau raster beserta titik dan bounding box hasil deteksi.

Contoh hasil pengujian pada satu citra orthomosaic (13.573 x 13.621 px): 401 tile terkirim ke Roboflow, dengan total 3.357 pohon sawit terhitung setelah deduplikasi.

## Catatan & Tips

* **`CONF_THRESHOLD`**: naikkan jika banyak deteksi palsu (*false positive*), turunkan jika banyak pohon yang terlewat.
* **`DEDUP_DIST_M`**: sesuaikan dengan jarak tanam sawit riil di lapangan agar dua pohon yang memang berdekatan tidak keliru dianggap satu pohon.
* Bounding box dihitung dari ukuran prediksi Roboflow di tiap tile, lalu dikonversi ke koordinat peta mengikuti transform tile masing-masing - sehingga ukurannya mengikuti skala asli raster, bukan ukuran tile 640px.
* Jika raster masih berreferensi **geografis (WGS84)**, sebaiknya direproyeksi ke UTM (lewat QGIS, `gdalwarp`, atau `rasterio.warp`) sebelum diproses, agar perhitungan jarak dan luas lebih akurat. Notebook tetap berjalan untuk raster geografis (menggunakan pendekatan konversi derajat ke meter), tetapi UTM lebih presisi.
* Format **ECW** tidak digunakan karena bersifat proprietary (membutuhkan SDK berbayar Hexagon/Erdas yang tidak tersedia di GDAL open-source). Alternatif open-source yang setara tujuannya untuk arsip/serving raster resolusi penuh adalah **Cloud Optimized GeoTIFF (COG)**, yang dapat dibuat lewat `gdal_translate -of COG input.tif output_cog.tif`.
* Peta interaktif menggunakan pratinjau yang di-*downsample* (maksimal 2000px sisi terpanjang) agar ringan di browser; cukup untuk overview, bukan untuk analisis piksel presisi.
* Jika permintaan ke Roboflow terus gagal (timeout dan sejenisnya), periksa kuota API di dashboard Roboflow, atau kemungkinan jaringan kampus/kantor memblokir domain `detect.roboflow.com`.
* **Keamanan API key**: jangan menanam API key Roboflow langsung di notebook yang akan dibagikan/dipublikasikan. Simpan lewat **Colab Secrets** (ikon kunci di sidebar kiri), lalu panggil dengan `google.colab.userdata.get('ROBOFLOW_API_KEY')`.

## Model

Deteksi menggunakan model **Palm Oil Detection and Counting** dari [Roboflow Universe](https://universe.roboflow.com), dikembangkan oleh Prasadnr (`model_id: palm-oil-detection-and-counting/5`), diakses melalui Roboflow Hosted Inference API.

## Penulis

**Jariyan Arifudin**
Geografi Lingkungan, Universitas Gadjah Mada (UGM)

## Lisensi

Kode ini dapat didistribusikan di bawah **MIT License** - sesuaikan berkas `LICENSE` pada repositori dengan preferensi Anda. Model deteksi yang digunakan mengikuti ketentuan lisensi dari [Roboflow Universe](https://universe.roboflow.com).
