# LintasSim — Simulasi Sensor E-Tilang Virtual

**LintasSim** adalah simulasi sistem E-Tilang dan *speed trap* ala proyek mikrokontroler Arduino yang berjalan sepenuhnya di browser. Alih-alih merakit board fisik, kabel jumper, LED, dan sensor inframerah sungguhan, kamu cukup menggerakkan kursor mouse (di PC) atau menyapukan jari (di HP) melintasi garis sensor di layar.

Seluruh kode—HTML, CSS, dan JavaScript—diringkas ke dalam **satu file tunggal (`index.html`)** tanpa framework, tanpa proses *build*, dan bisa langsung dibuka bahkan saat tidak ada koneksi internet.

---

## Kenapa Proyek Ini Dibuat?

Banyak orang yang sedang belajar elektronika dasar atau mengerjakan tugas sistem tertanam (*embedded systems*) sering terbentur hal-hal teknis sebelum logikanya sempat diuji: board belum datang, sensor IR bermasalah, atau sekadar butuh media demo yang mudah dibuka di HP saat presentasi.

LintasSim hadir untuk menjembatani itu:
* **Uji logika tanpa hardware:** Menguji aturan lampu lalu lintas, batas kecepatan, serta pemicu pelanggaran tanpa perlu merangkai komponen fisik.
* **Demonstrasi praktis lintas perangkat:** Cukup buka satu file HTML di laptop atau HP, gerakkan kursor/jari, dan seluruh indikator langsung merespons secara *real-time*.
* **Referensi arsitektur non-blocking:** Menunjukkan bagaimana pola `millis()` milik Arduino diterjemahkan ke JavaScript modern agar timer lampu, sensor, suara, dan animasi berjalan berbarengan tanpa saling menghambat.

---

## Apa Saja yang Bisa Dilakukan?

* **Pelacakan Mouse & Layar Sentuh (PC & HP):** Mendeteksi perangkat input secara otomatis. Di HP, area jalan sudah dikunci agar sapuan jari tidak ikut menggeser (*scroll*) halaman.
* **Deteksi Dua Jenis Pelanggaran:**
  * **Ngebut (*Overspeed*):** Melintas melebihi batas kecepatan yang diatur lewat *slider* (25–140 km/jam).
  * **Terobos Lampu Merah:** Melintas ketika status lampu sedang merah.
  * **Tilang Ganda:** Terdeteksi otomatis jika kedua pelanggaran terjadi di saat bersamaan.
* **Grafik Kecepatan Langsung (*Sparkline*):** Memantau perubahan kecepatan per 60 milidetik lengkap dengan garis putus-putus batas aman.
* **Kontrol Lampu Manual & Otomatis:** Siklus lampu berjalan otomatis (Hijau 5 detik → Kuning 2 detik → Merah 5 detik), tetapi bisa langsung ditahan ke warna tertentu untuk keperluan uji coba.
* **Monitor Pin Board Virtual:** Menampilkan kondisi logika `HIGH` / `LOW` untuk pin LED (`D2`, `D3`, `D4`), sensor IR (`D7`), dan buzzer (`D8`).
* **Suara Piezo Buzzer Virtual:** Menghasilkan bunyi *beep* aman dan alarm dua nada saat pelanggaran langsung dari browser.
* **Ekspor Riwayat ke CSV:** Seluruh catatan melintas (waktu, status, keterangan, kecepatan, batas, dan warna lampu) bisa diunduh dalam format `.csv` untuk dibuka di Excel atau Google Sheets.

---

## Bedah Cara Kerja Script (Sampai ke Akar)

Kode di dalam `index.html` disusun dengan pendekatan berorientasi objek (*Object-Oriented Programming*) yang terbagi ke dalam 4 kelas utama agar masing-masing punya tanggung jawab jelas:

| Kelas | Tanggung Jawab Utama |
| :--- | :--- |
| `TrafficLight` | Mengatur pergantian fase lampu lalu lintas berbasis waktu non-blocking. |
| `Buzzer` | Mensintesis suara piezo buzzer menggunakan Web Audio API. |
| `Sparkline` | Menggambar kurva kecepatan langsung pada canvas mini di kartu grafik. |
| `TrafficSim` | Menangani input mouse/sentuhan, matematika sensor, rendering jalan, statistik, dan ekspor CSV. |

Berikut penjelasan teknis di balik setiap mekanismenya:

### 1. Meniru `loop()` dan `millis()` Arduino
Di Arduino asli, memakai `delay(5000)` akan membekukan seluruh program sehingga sensor tidak bisa membaca apa pun selama lampu menyala. Di LintasSim, mekanisme `millis()` ditiru menggunakan `performance.now()` di dalam siklus `requestAnimationFrame`:

1. Setiap *frame* (~60 kali per detik), `TrafficLight.update(now)` membandingkan selisih waktu saat ini dengan waktu mulai fase (`now - this.since`).
2. Jika selisihnya sudah mencapai durasi fase (misalnya $5000\text{ ms}$ untuk hijau), indeks fase bergeser ke warna berikutnya tanpa menghentikan pembacaan sensor maupun animasi.

### 2. Menghitung Kecepatan Gerakan (*Sliding Window*)
Mengukur kecepatan hanya dari dua *frame* terakhir sering membuat angkanya melonjak tidak stabil karena selisih waktu antar-event browser sangat tipis. LintasSim mengatasinya dengan metode **jendela waktu geser ($80\text{ ms}$)**:

* Setiap kali kursor/jari bergerak, koordinat $(x, y)$ dan stempel waktu $t$ dimasukkan ke larik `samples`.
* Sampel yang usianya lebih tua dari $80\text{ ms}$ langsung dibuang.
* Jarak tempuh total dihitung dengan menjumlahkan jarak Euclidean antar titik berurutan:
  $$d_i = \sqrt{(x_i - x_{i-1})^2 + (y_i - y_{i-1})^2}$$
* Kecepatan dalam satuan piksel per detik lalu dikonversi ke skala simulasi $\text{km/jam}$ dengan faktor kalibrasi $k = 0.075$:
  $$v = \left(\frac{\sum d_i}{\Delta t} \times 1000\right) \times 0.075$$
* Saat gerakan berhenti lebih dari $120\text{ ms}$, kecepatan melambat secara bertahap (`this.spd *= 0.75`) hingga kembali ke `0`, meniru perlambatan alami.

### 3. Algoritma Anti-Tembus Garis (*Line Segment Intersection*)
Masalah klasik pada deteksi garis di browser adalah *tunneling*: ketika mouse atau jari disapukan sangat cepat, posisi pada *frame* A berada jauh di atas garis sensor ($y_1 < y_s$), lalu pada *frame* B sudah berada jauh di bawahnya ($y_2 > y_s$) tanpa pernah menyentuh piksel tepat di $y_s$.

LintasSim tidak mengecek apakah kursor "sedang menyentuh garis", melainkan mengecek apakah **segmen lintasan dari titik $A(x_1, y_1)$ ke $B(x_2, y_2)$ memotong garis horizontal sensor $y_s$**:

1. Pastikan posisi vertikal berpindah sisi: $(y_1 < y_s \text{ dan } y_2 \ge y_s)$ atau sebaliknya.
2. Hitung titik potong horizontal $x_i$ menggunakan interpolasi linear:
   $$x_i = x_1 + \frac{y_s - y_1}{y_2 - y_1}(x_2 - x_1)$$
3. Periksa apakah titik potong $x_i$ jatuh di dalam lebar badan jalan:
   $$x_{\text{jalan}} \le x_i \le x_{\text{jalan}} + w_{\text{jalan}}$$
4. Terapkan jeda baca (*cooldown / debounce*) selama $520\text{ ms}$ agar getaran kecil di sekitar garis tidak memicu tilang berkali-kali untuk satu kali lewat.

### 4. Penanganan Input PC dan HP (`PointerEvent`)
Daripada menulis dua logika terpisah untuk `MouseEvent` dan `TouchEvent`, script memakai standar **`PointerEvent`** yang menyatukan mouse, jari, dan stylus:
* Pada canvas `#road`, properti CSS `touch-action: none` digabungkan dengan `e.preventDefault()` pada `touchstart` dan `touchmove` supaya sapuan jari vertikal tidak memicu *scroll* halaman atau *pull-to-refresh* di browser HP.
* `setPointerCapture(e.pointerId)` menjaga pembacaan tetap nyambung meski jari sedikit keluar dari tepi area jalan saat sedang menyapu cepat.
* Di PC, kursor cukup digerakkan masuk ke area jalan (`pointerenter`) tanpa perlu menahan klik. Di HP, pelacakan aktif selama jari menempel dan menggeser layar (`pointerdown` → `pointermove` → `pointerup`).

### 5. Sintesis Suara Buzzer Tanpa File Audio
Kelas `Buzzer` memanfaatkan **Web Audio API** bawaan browser:
* Saat melintas aman, dibuat gelombang sinus (`sine`) $1240\text{ Hz}$ selama $45\text{ ms}$.
* Saat terjadi pelanggaran, dibuat gelombang kotak (`square`)—yang karakter bunyinya mirip piezo buzzer asli—dengan dua nada berurutan: $880\text{ Hz}$ ($110\text{ ms}$) disusul $620\text{ Hz}$ ($170\text{ ms}$).
* `GainNode` dipakai untuk menurunkan volume secara eksponensial di akhir nada agar tidak muncul bunyi "klik" kasar di speaker.

---

## Pemetaan Pin Virtual (Jika Ingin Dipindah ke Arduino Asli)

Jika logika simulasi ini suatu saat ingin dirakit ke **Arduino Uno (ATmega328P)** sungguhan, berikut pemetaan pin yang digunakan di dashboard:

| Pin Digital | Mode | Komponen Fisik Padanan | Fungsi |
| :--- | :--- | :--- | :--- |
| `D2` | `OUTPUT` | LED Merah + Resistor $220\,\Omega$ | Indikator berhenti |
| `D3` | `OUTPUT` | LED Kuning + Resistor $220\,\Omega$ | Indikator hati-hati |
| `D4` | `OUTPUT` | LED Hijau + Resistor $220\,\Omega$ | Indikator jalan |
| `D7` | `INPUT` | Modul Sensor IR / Laser Photodiode | Mendeteksi objek melintasi garis |
| `D8` | `OUTPUT` | Active / Passive Piezo Buzzer | Bunyi peringatan saat pelanggaran |

---

## Cara Menjalankan

Karena semua kode sudah berada di dalam satu file, kamu punya tiga cara termudah untuk menjalankannya:

1. **Langsung di PC:** Klik dua kali file `index.html`, otomatis terbuka di browser (Chrome, Edge, Firefox, atau Safari).
2. **Langsung di HP:** Simpan `index.html` di penyimpanan HP, lalu buka lewat file manager menggunakan Chrome/browser bawaan.
3. **Lewat Server Lokal (untuk tes HP via Wi-Fi yang sama):**
   ```bash
   npx serve .
   ```
   Lalu buka alamat IP lokal yang muncul di terminal lewat browser HP.

---

## Developer

Proyek ini dirancang dan dikembangkan oleh **Zann**.

Fokus utama pengembangannya adalah membangun alat bantu simulasi dan antarmuka web yang ringan, bersih, berjalan cepat bahkan di perangkat berspesifikasi terbatas, serta mudah dipahami strukturnya oleh siapa saja yang ingin memodifikasi atau mengembangkannya lebih lanjut.