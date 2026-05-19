# Dokumentasi Lengkap Analisis PID
## Sistem 34_LC_013 — Level Steam Drum Controller, WHB_FEEDWATER
### PT Pusri IB

---

# BAGIAN 1: GOOGLE COLAB

---

## 1.1 Latar Belakang dan Data

### Data yang Digunakan

Data berasal dari historian DCS FoxView PT Pusri IB, file `Combined_Default_Data.xlsx`:

| Kolom | Nilai Tipikal | Keterangan |
|---|---|---|
| `DateTime` | Feb 1–28, 2026 | Timestamp, interval **600 detik (10 menit)** |
| `SPT` | 53% (konstan) | Setpoint level steam drum |
| `MEAS` | 50.1–54.8% | Process Variable — level aktual steam drum |
| `OUT` | 59.2–70.3% | Output controller — bukaan valve feedwater |

**Total: 4032 baris data, 4 file sumber (Default1–4.txt)**

### Kondisi Sistem Saat Pengambilan Data

Controller aktif dalam mode **closed-loop** sepanjang waktu. Artinya:
- Ketika MEAS naik → controller menurunkan OUT
- Ketika MEAS turun → controller menaikkan OUT
- SPT tetap di 53% — tidak ada perubahan setpoint

---

## 1.2 Eksplorasi Data (EDA)

### Statistik Deskriptif

| | MEAS | OUT |
|---|---|---|
| **Mean** | 53.02% | 65.69% |
| **Std** | 0.37% | 1.62% |
| **Min** | 50.11% | 59.25% |
| **Max** | 54.82% | 70.29% |

### Temuan Kunci: Korelasi Negatif

$$r_{OUT,MEAS} = -0.1796$$

**Mengapa korelasi negatif?**

Ini adalah efek langsung dari **feedback control yang aktif**:

```
MEAS naik (level tinggi)
    → controller turunkan OUT (valve menutup)
    → korelasi OUT-MEAS tampak negatif
```

Bukan berarti valve reverse-acting. Ini justru bukti controller bekerja dengan baik.

**Konsekuensi untuk identifikasi:**

Identifikasi model langsung dari data closed-loop akan menghasilkan parameter yang **bias** karena sinyal OUT dan MEAS tidak independen. Inilah alasan mengapa diperlukan pendekatan dua langkah: ARX → PRBS.

---

## 1.3 Model FOPDT — Konsep Dasar

Sebelum masuk ke proses identifikasi, penting memahami apa itu FOPDT dan mengapa digunakan.

### Definisi FOPDT

FOPDT (First Order Plus Dead Time) adalah model matematis proses industri yang paling umum digunakan. Transfer function-nya dalam domain Laplace:

$$G(s) = \frac{K_p \cdot e^{-\theta s}}{\tau s + 1}$$

### Parameter dan Maknanya

**Kp — Process Gain**

$$K_p = \frac{\Delta \text{Output (MEAS)}}{\Delta \text{Input (OUT)}}$$

- Menunjukkan **seberapa besar** PV berubah untuk setiap 1% perubahan valve
- Kp = 0.8616 artinya: setiap valve bertambah 1%, level naik 0.8616%
- Dimensi: %/% (adimensional untuk proses level)

**τ (tau) — Time Constant**

- Waktu yang diperlukan PV untuk mencapai **63.2%** dari nilai akhirnya setelah step input
- τ = 14471 s = 241 menit artinya: proses sangat lambat
- Semakin besar τ, semakin **lambat** respons proses
- Menentukan seberapa "malas" proses merespons perubahan valve

**θ (theta) — Dead Time**

- Waktu tunda sebelum proses **mulai merespons sama sekali**
- θ = 1978 s = 33 menit artinya: setelah valve bergerak, level baru mulai berubah setelah 33 menit
- Disebabkan oleh: transport delay pipa, waktu mixing, sensor lag
- Semakin besar θ, semakin **sulit** sistem dikontrol

**Rasio θ/τ — Controllability Index**

$$\frac{\theta}{\tau} = \frac{1978}{14471} = 0.137$$

- < 0.3 → proses mudah dikontrol ✅
- 0.3–0.7 → sedang
- > 0.7 → sulit dikontrol

Sistem ini termasuk kategori **mudah dikontrol** meskipun dead time-nya besar.

### Representasi Diskrit FOPDT

Untuk simulasi numerik, FOPDT dikonversi ke persamaan diferensial diskrit (Euler forward):

$$y[k] = y[k-1] + \frac{K_p \cdot u[k-d] - y[k-1]}{\tau} \cdot \Delta t$$

Dimana:
- $y[k]$ = output proses (MEAS) pada langkah ke-k
- $u[k-d]$ = input valve pada langkah ke-(k-d), yaitu sudah tertunda d langkah
- $d = \theta / \Delta t$ = jumlah langkah dead time
- $\Delta t$ = interval sampling = 600 detik

**Dead time** diimplementasikan sebagai **buffer FIFO** (First In First Out):
- Sinyal valve disimpan dalam antrian sepanjang d langkah
- Sinyal yang baru masuk baru "dirasakan" proses setelah d langkah

---

## 1.4 Langkah 1: Estimasi Awal via ARX

### Mengapa ARX Diperlukan?

Untuk menjalankan simulasi PRBS, kita membutuhkan **estimasi awal parameter proses** sebagai plant inisial. ARX digunakan untuk tujuan ini — meskipun hasilnya tidak akurat, estimasi ini cukup sebagai titik awal.

> **Penting:** Hasil ARX **bukan model final**. ARX hanya berfungsi sebagai estimasi awal untuk PRBS.

### Konsep Model ARX

ARX (AutoRegressive with eXogenous input) adalah model diskrit linier:

$$y[k] = a \cdot y[k-1] + b \cdot u[k-d] + e[k]$$

Dimana:
- $y[k]$ = MEAS pada waktu ke-k
- $y[k-1]$ = MEAS satu langkah sebelumnya (autoregressive term)
- $u[k-d]$ = OUT dengan delay d langkah (exogenous input)
- $a, b$ = koefisien yang dicari
- $e[k]$ = residual error
- $d$ = delay yang dicoba dari 0 sampai 15 langkah

### Darimana Nilai a dan b Berasal?

Untuk setiap nilai delay d, dibentuk matriks regresi:

$$\mathbf{A} = \begin{bmatrix} y[d+1] & u[1] \\ y[d+2] & u[2] \\ \vdots & \vdots \\ y[N] & u[N-d] \end{bmatrix}, \quad \mathbf{y} = \begin{bmatrix} y[d+2] \\ y[d+3] \\ \vdots \\ y[N+1] \end{bmatrix}$$

Solusi **Least Squares** (meminimalkan jumlah kuadrat error):

$$\begin{bmatrix} a \\ b \end{bmatrix} = (\mathbf{A}^T \mathbf{A})^{-1} \mathbf{A}^T \mathbf{y}$$

Delay terbaik dipilih berdasarkan **R² tertinggi**:

$$R^2 = 1 - \frac{\sum(y - \hat{y})^2}{\sum(y - \bar{y})^2}$$

### Konversi Koefisien ARX → Parameter FOPDT

Dari model diskrit FOPDT yang dilinearkan:

$$y[k] = e^{-\Delta t/\tau} \cdot y[k-1] + K_p(1 - e^{-\Delta t/\tau}) \cdot u[k-d]$$

Maka:

$$a = e^{-\Delta t/\tau} \implies \boxed{\tau = \frac{-\Delta t}{\ln(a)}}$$

$$b = K_p(1-a) \implies \boxed{K_p = \frac{b}{1-a}}$$

$$\boxed{\theta = d \cdot \Delta t}$$

**Syarat valid:** $0 < a < 1$ (proses stabil first-order)

### Hasil ARX

| Parameter | Nilai |
|---|---|
| Delay terbaik | d = 3 steps = 30 menit |
| Koefisien a | 0.952091 |
| Koefisien b | 0.038651 |
| **R²** | **-0.106 (negatif!)** |
| Kp | 0.8068 |
| τ | 12221 s (203.7 menit) |
| θ | 1800 s (30 menit) |

### Mengapa R² Negatif?

R² negatif berarti model lebih buruk dari sekadar prediksi rata-rata. Ini terjadi karena:

1. Data closed-loop menyebabkan **korelasi bias** antara OUT dan MEAS
2. ARX mengasumsikan OUT bebas dari MEAS (open-loop) — asumsi ini dilanggar
3. Sinyal OUT dan MEAS saling mempengaruhi, bukan hubungan sebab-akibat yang jelas

**Ini bukan kegagalan** — ini konfirmasi bahwa data closed-loop memang tidak cocok untuk identifikasi ARX langsung. Nilai parameter (Kp=0.8068, τ=12221s, θ=1800s) digunakan sebagai **estimasi kasar** untuk plant inisial PRBS.

---

## 1.5 Langkah 2: Identifikasi FOPDT via PRBS Injection

### Mengapa PRBS?

PRBS (Pseudo-Random Binary Sequence) adalah solusi untuk masalah data closed-loop. Dengan menginjeksikan sinyal PRBS ke output controller, kita "memaksa" proses untuk merespons sinyal yang **tidak berkorelasi dengan feedback loop**, sehingga identifikasi menjadi lebih akurat.

### Konsep PRBS

PRBS adalah sinyal biner deterministik yang berpindah antara dua nilai (+A dan −A) secara acak semu. Karakteristik utamanya:

- **Broadband spectrum** — mengandung banyak frekuensi sekaligus → mengeksitasi semua dinamika proses
- **Amplitudo terbatas** — tidak terlalu mengganggu operasi
- **Deterministik** — bisa direncanakan dan direproduksi
- **Tidak berkorelasi dengan feedback** — memutus bias closed-loop

### Cara Kerja LFSR (Linear Feedback Shift Register)

PRBS dibangkitkan menggunakan LFSR orde n=7:

```
Inisialisasi: state = [1, 1, 1, 1, 1, 1, 1]  (7 bit, semua 1)

Untuk setiap langkah:
    bit_output = state[6]                    (bit paling kanan)
    new_bit    = state[6] XOR state[5]       (feedback tap posisi 7,6)
    state      = [new_bit, state[0..5]]      (geser register)
    
    output = +A jika bit_output = 1
    output = -A jika bit_output = 0
```

Panjang sequence: $2^7 - 1 = 127$ clock periods (tidak berulang dalam satu siklus)

### Parameter Desain PRBS

| Parameter | Nilai | Rumus/Alasan |
|---|---|---|
| n bits | 7 | Panjang = 2⁷−1 = 127 |
| Clock period | 2 steps = 20 menit | Clock ≥ τ/10 agar proses bisa merespons |
| Amplitudo | ±3.23% | = 2 × std(OUT) = 2 × 1.62% |
| Durasi total | 254 steps = 42.3 jam | >> 3τ = 3 × 241 min = 723 menit |
| OUT range | 62.46% – 68.92% | Dalam batas operasi normal |

**Mengapa amplitudo = 2×std(OUT)?**
- Cukup besar untuk SNR (Signal-to-Noise Ratio) > 5
- Tidak terlalu besar sehingga tidak mengganggu operasi
- Masih dalam range normal operasi valve

### Dari ARX ke PRBS: Hubungannya

Parameter ARX (Kp=0.8068, τ=12221s, θ=1800s) digunakan sebagai **plant inisial** untuk mensimulasikan respons proses terhadap PRBS:

```
Parameter ARX (estimasi kasar)
    ↓
Plant inisial FOPDT
    ↓ diberi sinyal PRBS
Respons simulasi (y_prbs)
    ↓
Identifikasi DE → parameter FOPDT final
```

**Persamaan simulasi plant:**

$$y[k] = y[k-1] + \frac{K_p \cdot u_{PRBS}[k-d] - y[k-1]}{\tau} \cdot \Delta t + \varepsilon[k]$$

Dimana $\varepsilon[k] \sim \mathcal{N}(0, \sigma^2)$ adalah noise Gaussian (std=0.08) untuk mensimulasikan kondisi nyata.

### Identifikasi Parameter: Differential Evolution (DE)

Dari pasangan data (sinyal PRBS, respons plant), parameter FOPDT dicari dengan **meminimalkan MSE**:

$$\text{MSE}(K_p, \tau, \theta) = \frac{1}{N} \sum_{k=1}^{N} \left(y_{aktual}[k] - y_{model}(K_p, \tau, \theta)[k]\right)^2$$

**Mengapa Differential Evolution?**

| Kriteria | DE | Gradient-based (L-BFGS) |
|---|---|---|
| Konvergensi global | ✅ Tidak terjebak local minimum | ❌ Bisa terjebak |
| Perlu gradien | ❌ Tidak perlu | ✅ Perlu |
| Parameter skala beda | ✅ Robust | ❌ Sensitif skala |
| Fungsi non-smooth | ✅ Bisa | ❌ Masalah |

**Konfigurasi DE yang digunakan:**

| Setting | Nilai | Keterangan |
|---|---|---|
| Bounds Kp | [0.01, 5.0] | Range gain proses yang realistis |
| Bounds τ | [60, 36000] s | 1 menit hingga 10 jam |
| Bounds θ | [0, 7200] s | 0 hingga 2 jam |
| Populasi | 20 individu | Jumlah solusi kandidat per generasi |
| Max iterasi | 600 | Batas iterasi optimasi |
| Mutation | [0.5, 1.5] | Faktor mutasi |
| Recombination | 0.9 | Probabilitas crossover |
| Polish | True | Fine-tuning akhir dengan L-BFGS-B |

**Cara kerja DE (simpel):**
1. Bangkitkan 20 kandidat parameter secara acak dalam bounds
2. Untuk setiap kandidat, hitung MSE
3. Mutasi: kombinasikan kandidat dengan kandidat lain
4. Seleksi: pertahankan kandidat dengan MSE lebih kecil
5. Ulangi hingga konvergen atau max iterasi

### Hasil Identifikasi PRBS

| Parameter | Nilai | Keterangan |
|---|---|---|
| **Kp** | **0.8616** | Gain proses final |
| **τ** | **14471 s (241.2 menit)** | Time constant final |
| **θ** | **1978 s (33.0 menit)** | Dead time final |
| MSE | 0.038519 | Error kuadrat rata-rata |
| **R²** | **0.8685** ✅ | Model menjelaskan 86.85% variansi |

**Interpretasi R² = 0.8685:**
Model FOPDT hasil PRBS mampu mereproduksi 86.85% variasi respons proses. Ini jauh lebih baik dari ARX (R²=-0.106) dan cukup akurat untuk keperluan tuning PID.

---

## 1.6 Langkah 3: Perhitungan Ku dan Pu

### Mengapa Perlu Ku dan Pu?

Metode Tyreus-Luyben menggunakan **Ku (Ultimate Gain)** dan **Pu (Ultimate Period)** sebagai dasar perhitungan parameter PID. Ku dan Pu mencerminkan batas kestabilan marginal sistem.

Ku = gain controller tertinggi yang masih bisa membuat sistem berosilasi terus-menerus (tidak divergen).
Pu = periode osilasi pada kondisi tersebut.

### Pendekatan: Frequency Response Analysis

Alih-alih mencari Ku dan Pu secara eksperimen (berbahaya di sistem nyata), kita hitung secara **analitik dari model FOPDT** via analisis frekuensi.

### Derivasi Persamaan

**Transfer function FOPDT pada frekuensi ω** (substitusi $s = j\omega$):

$$G(j\omega) = \frac{K_p \cdot e^{-j\omega\theta}}{j\omega\tau + 1}$$

**Magnitude:**

$$|G(j\omega)| = \frac{K_p}{\sqrt{1 + (\omega\tau)^2}}$$

**Phase:**

$$\angle G(j\omega) = -\arctan(\omega\tau) - \omega\theta$$

**Kondisi osilasi marginal (Nyquist criterion):**

Sistem berada pada batas kestabilan ketika:

$$|G(j\omega_u)| \cdot K_u = 1 \quad \text{dan} \quad \angle G(j\omega_u) = -180°$$

**Dari kondisi phase = -180°:**

$$-\arctan(\omega_u \tau) - \omega_u \theta = -\pi$$

$$\boxed{f(\omega_u) = -\arctan(\omega_u \tau) - \omega_u \theta + \pi = 0}$$

Persamaan ini **tidak bisa diselesaikan secara analitik**, maka diselesaikan secara **numerik** menggunakan Newton-Raphson (`scipy.fsolve`) dengan tebakan awal $\omega_0 = 0.001$ rad/s.

**Setelah ωu diperoleh:**

$$\boxed{P_u = \frac{2\pi}{\omega_u}}$$

Dari kondisi gain:

$$K_u = \frac{1}{|G(j\omega_u)|} = \frac{\sqrt{1 + (\omega_u\tau)^2}}{K_p}$$

$$\boxed{K_u = \frac{\sqrt{1 + (\omega_u\tau)^2}}{K_p}}$$

### Hasil Perhitungan

| Parameter | Nilai |
|---|---|
| ωu | 0.000836 rad/s |
| **Ku** | **14.0849** |
| **Pu** | **7518.0 s (125.3 menit)** |

---

## 1.7 Langkah 4: Tuning Tyreus-Luyben

### Latar Belakang

Tyreus dan Luyben (1992) mengembangkan metode tuning berdasarkan Ku dan Pu yang menghasilkan respons lebih **konservatif dan stabil** dibanding Ziegler-Nichols — cocok untuk proses industri yang tidak toleran overshoot.

### Perbandingan Tyreus-Luyben vs Ziegler-Nichols

| Parameter | Ziegler-Nichols | **Tyreus-Luyben** | Efek |
|---|---|---|---|
| **Kc** | Ku / 1.7 | **Ku / 2.20** | TL lebih kecil → lebih stabil |
| **Ti** | Pu / 2 | **2.2 × Pu** | TL lebih besar → integral lebih lambat |
| **Td** | Pu / 8 | **Pu / 6.3** | TL lebih besar → derivative lebih kuat |

### Rumus Tyreus-Luyben

$$\boxed{K_c = \frac{K_u}{2.20} = \frac{14.0849}{2.20} = 6.4022}$$

$$\boxed{T_i = 2.2 \cdot P_u = 2.2 \times 7518 = 16540 \text{ s}}$$

$$\boxed{T_d = \frac{P_u}{6.3} = \frac{7518}{6.3} = 1193 \text{ s}}$$

### Parameter Existing PIDA (dari FoxView 34_LC_013)

Dari layar FoxView DCS yang terbaca:

| Parameter FoxView | Nilai | Konversi | Nilai PID |
|---|---|---|---|
| PBAND | 80 | Kc = 100/PBAND | **Kc = 1.25** |
| IR | 100 menit/repeat | Ti = IR × 60 | **Ti = 6000 s** |
| DERIV | 0 | Td = 0 | **Td = 0 (PI only)** |

### Tabel Perbandingan Parameter

| | Existing PIDA | Tyreus-Luyben |
|---|---|---|
| **Tipe** | PI | PID |
| **Kc** | 1.25 | 6.4022 |
| **Ti** | 6000 s (100 menit) | 16540 s (275.7 menit) |
| **Td** | 0 s | 1193 s (19.9 menit) |

---

## 1.8 Langkah 5: Simulasi Closed-Loop di Colab

### Algoritma PID Standard Form

$$u(t) = K_c \left[ e(t) + \frac{1}{T_i}\int_0^t e\,d\tau + T_d\frac{de}{dt} \right]$$

Dalam bentuk diskrit:

$$u[k] = K_c \left[ e[k] + \frac{\Delta t}{T_i} \sum_{i=0}^{k} e[i] + \frac{T_d}{\Delta t}(e[k] - e[k-1]) \right]$$

### Fitur Implementasi

**Anti-Windup:**

Mencegah integral terus terakumulasi saat valve saturasi:

```python
if not (0 < valve < 100):
    integral -= error * dt  # batalkan akumulasi
```

**Slew Rate Limiter:**

Valve fisik tidak bisa bergerak lebih dari 10%/menit:

```python
max_delta = 10.0 * dt / 60
valve = prev_valve + clip(valve - prev_valve, -max_delta, max_delta)
```

**Dead Time Buffer (FIFO):**

```python
delay_steps = theta / dt
buffer = deque([0.0] * delay_steps)
u_delayed = buffer[0]
buffer.append(valve - mean_out)
```

### Metrik Performa

| Metrik | Rumus | Interpretasi |
|---|---|---|
| **ISE** | $\sum e[k]^2 \cdot \Delta t$ | Penalti besar untuk error besar |
| **IAE** | $\sum \|e[k]\| \cdot \Delta t$ | Penalti proporsional |
| **ITAE** | $\sum t[k] \cdot \|e[k]\| \cdot \Delta t$ | Penalti untuk error yang bertahan lama |
| **Overshoot** | $(PV_{max} - SP)/SP \times 100\%$ | Seberapa jauh PV melampaui SP |
| **Rise Time** | $t_{90\%} - t_{10\%}$ | Kecepatan respons awal |
| **Settling Time** | $\max\{t : \|e\| > 2\% \cdot \Delta_{SP}\}$ | Waktu hingga sistem stabil |
| **SS Error** | $\|\bar{e}\|_{90\%-100\%}$ | Sisa error di steady state |

### Hasil Metrik Performa

| Metrik | Existing PIDA | TL dari PRBS | Improvement |
|---|---|---|---|
| **ISE** | 699.54 | 353.46 | ↓ **49.5%** |
| **IAE** | 3670.16 | 1800.51 | ↓ **50.9%** |
| **ITAE** | 41,259,023 | 13,742,409 | ↓ **66.7%** |
| **Overshoot** | 0.23% | 0.10% | ↓ lebih stabil |
| **Rise Time** | 7080 s | 2280 s | ↓ **67.8%** |
| **SS Error** | 0.08471 | 0.01975 | ↓ **76.7%** |

---

# BAGIAN 2: MATLAB SIMULINK

---

## 2.1 Struktur Model Simulink

Model Simulink terdiri dari **dua loop paralel** yang berbagi sinyal setpoint dan disturbance yang sama:

```
                    ┌─────────────────────────────────────────────────┐
                    │              LOOP EXISTING PIDA                  │
[SP] ──┬──────────▶ [Sum] ──▶ [PID] ──▶ [SAT] ──▶ [SumDist] ──▶ [DT] ──▶ [FOPDT] ──▶ [SumOffset] ──▶ [Scope]
       │              ▲─────────────────────────────────────────────────────────────────────────────────┘
       │
       │           ┌─────────────────────────────────────────────────┐
       │           │              LOOP TYREUS-LUYBEN                  │
       └─────────▶ [Sum] ──▶ [PID] ──▶ [SAT] ──▶ [SumDist] ──▶ [DT] ──▶ [FOPDT] ──▶ [SumOffset] ──▶ [Scope]
                    ▲─────────────────────────────────────────────────────────────────────────────────────┘

[Disturbance] ──────────────────────────────┘ (ke kedua SumDist)
[Mux] ◀── kedua output ──▶ [Scope_Comparison]
```

---

## 2.2 Penjelasan Detail Setiap Blok

### Blok 1: SP (Step — Setpoint)

**Fungsi:** Membangkitkan sinyal setpoint yang berubah dari kondisi awal ke target.

**Parameter:**
| Parameter | Nilai | Keterangan |
|---|---|---|
| Step time | 600 s | Setpoint berubah di t=600s |
| Initial value | 52.61% | Kondisi awal PV dari FoxView |
| Final value | 53.00% | Target level steam drum |

**Representasi:** Sinyal referensi yang ingin dicapai controller.

**Mengapa 52.61?** Karena itu nilai MEAS aktual yang terbaca di FoxView saat data diambil — kondisi awal sistem yang realistis.

---

### Blok 2: Sum (Hitung Error)

**Fungsi:** Menghitung error = SP − PV (selisih antara setpoint dan nilai aktual).

**Parameter:**
- Signs: `+-` (SP masuk port +, PV masuk port −)

**Rumus:**
$$e[k] = SP[k] - PV[k]$$

**Mengapa penting?** Error inilah yang menjadi input PID controller. Semakin besar error, semakin besar aksi koreksi controller.

---

### Blok 3: PID Controller

**Fungsi:** Menghitung sinyal kontrol berdasarkan error menggunakan algoritma PID.

**Parameter Existing PIDA:**
| Parameter Simulink | Nilai | Keterangan |
|---|---|---|
| P | 1.25 | = Kc existing |
| I | 0.000208 | = Kc/Ti = 1.25/6000 |
| D | 0 | DERIV=0, mode PI |
| N | 100 | Filter coefficient derivative |

**Parameter Tyreus-Luyben:**
| Parameter Simulink | Nilai | Keterangan |
|---|---|---|
| P | 6.4022 | = Kc TL |
| I | 0.000387 | = Kc/Ti = 6.4022/16540 |
| D | 7638.2 | = Kc×Td = 6.4022×1193 |
| N | 100 | Filter coefficient |

**Mengapa I = Kc/Ti (bukan Ti)?**

Di Simulink, blok PID menggunakan **parallel form**:

$$u = P \cdot e + I \cdot \int e\,dt + D \cdot \frac{de}{dt}$$

Dimana:
- $P = K_c$
- $I = K_c / T_i$
- $D = K_c \cdot T_d$

Berbeda dengan **standard form** yang digunakan di persamaan PID konvensional.

**Anti-Windup:** Di Simulink, aktifkan via:
- Tab Output Saturation → Limit output: Upper=100, Lower=0
- Anti-windup method: back-calculation

---

### Blok 4: Saturation (Clamp Valve 0–100%)

**Fungsi:** Membatasi output PID agar tidak melebihi range fisik valve.

**Parameter:**
- Upper limit: 100%
- Lower limit: 0%

**Mengapa diperlukan?**

Valve fisik hanya bisa bergerak antara 0% (tutup penuh) dan 100% (buka penuh). Output PID yang tidak dibatasi bisa menghasilkan nilai negatif atau >100%.

**Hubungan dengan Anti-Windup:**

Saat valve saturasi (0% atau 100%), integral PID harus dihentikan agar tidak terus terakumulasi. Inilah fungsi anti-windup.

---

### Blok 5: SumDist (Penjumlahan Disturbance)

**Fungsi:** Menambahkan sinyal gangguan ke sinyal valve sebelum masuk proses.

**Parameter:**
- Signs: `++`

**Posisi:** Antara SAT dan DT (Dead Time)

**Mengapa di sini?**

Di industri nyata, disturbance pada level steam drum bisa berupa:
- Fluktuasi tekanan steam yang tiba-tiba
- Perubahan laju produksi
- Kebocoran kecil pada sistem

Gangguan ini **mempengaruhi proses** (bukan controller), sehingga dimodelkan sebagai gangguan pada **input proses** — yaitu setelah sinyal valve dan sebelum masuk ke model proses.

---

### Blok 6: Disturbance (Pulse Generator)

**Fungsi:** Membangkitkan sinyal gangguan periodik yang realistis.

**Parameter:**
| Parameter | Nilai | Keterangan |
|---|---|---|
| Amplitude | 3% | Besar gangguan valve |
| Period | 72000 s (20 jam) | Interval antar gangguan |
| Pulse width | 15% | Durasi gangguan = 3 jam |
| Phase delay | 43200 s (12 jam) | Gangguan mulai setelah system settle |

**Mengapa Pulse Generator (bukan Step)?**

Step hanya memberikan gangguan sekali. Pulse Generator mensimulasikan kondisi industri yang lebih realistis — gangguan datang berulang secara periodik.

---

### Blok 7: DT (Transport Delay — Dead Time θ)

**Fungsi:** Mensimulasikan dead time proses — waktu tunda sebelum perubahan valve dirasakan oleh proses.

**Parameter:**
- Delay time: **1978 s (33 menit)**

**Mengapa dead time terjadi di proses nyata?**

Untuk sistem feedwater steam drum:
- Waktu tempuh air dari valve ke drum melalui pipa
- Waktu mixing air baru dengan air yang sudah ada di drum
- Lag sensor level

**Efek dead time pada kontrol:**

Dead time membuat controller "buta" selama 33 menit — tidak bisa mendeteksi efek aksinya sendiri. Inilah yang membuat proses dengan dead time besar lebih sulit dikontrol.

---

### Blok 8: FOPDT (Transfer Function)

**Fungsi:** Mensimulasikan dinamika proses — bagaimana proses merespons perubahan valve.

**Parameter:**
- Numerator: `[0.8616]` ← Kp
- Denominator: `[14471 1]` ← [τ 1]

**Transfer function:**

$$G(s) = \frac{0.8616}{14471s + 1}$$

**Fisiknya:** Proses level steam drum merespons perubahan aliran feedwater secara first-order — perlahan naik/turun secara eksponensial menuju nilai baru.

**Mengapa τ = 14471s sangat besar?**

Steam drum memiliki volume air yang sangat besar → kapasitas thermal/hidrolik besar → perubahan level sangat lambat merespons perubahan aliran masuk.

---

### Blok 9: SumOffset (Tambah Bias 52.61%)

**Fungsi:** Mengkonversi output FOPDT dari deviasi ke nilai absolut PV.

**Parameter:**
- Constant: **52.61%** (mean MEAS dari data historis / kondisi awal)

**Mengapa diperlukan?**

FOPDT bekerja dalam **deviasi dari titik operasi** (nilai sekitar 0). Untuk mendapatkan nilai PV absolut (sekitar 52–54%), perlu ditambahkan nilai operasi awal.

$$PV_{absolut} = PV_{deviasi} + 52.61$$

**Ini juga yang menjadi sinyal feedback** ke port `−` blok Sum, sehingga controller melihat nilai PV yang benar (52–54%) bukan deviasi (0–1%).

---

### Blok 10: Mux

**Fungsi:** Menggabungkan dua sinyal (Existing dan TL) menjadi satu untuk ditampilkan dalam satu Scope.

**Parameter:**
- Number of inputs: 2

---

### Blok 11: Scope

Ada beberapa Scope dalam model:

| Scope | Input | Menampilkan |
|---|---|---|
| `EXISTING` | Output SumOffset Existing | PV loop Existing saja |
| `TL` | Output SumOffset TL | PV loop TL saja |
| `COMPARISON` | Mux (Existing + TL) | Perbandingan PV keduanya |
| `Scope_Valve` | Output SAT + offset 65.69 | Sinyal valve (display only) |

---

### Blok 12: Sum_Valve_Display (Display Only)

**Fungsi:** Menambahkan offset 65.69% ke sinyal valve untuk menampilkan nilai absolut valve.

**Parameter:**
- Signs: `++`
- Input 1: Output SAT (deviasi valve, sekitar 0–5%)
- Input 2: Constant 65.69 (mean OUT dari data historis)

**PENTING:** Blok ini hanya untuk **display** — tidak terhubung ke jalur kontrol utama. Jalur kontrol tetap menggunakan sinyal deviasi.

---

## 2.3 Alur Sinyal Lengkap

```
SINYAL SETPOINT
    SP = 52.61% → 53% (step di t=600s)
    │
    ▼
PERHITUNGAN ERROR
    e = SP - PV_absolut
    e = 53 - 52.61 = 0.39% (awal)
    │
    ▼
PID CONTROLLER
    u_raw = Kc × [e + (1/Ti)∫e + Td×de/dt]
    (sinyal deviasi, nilai kecil ~0–5%)
    │
    ▼
SATURATION
    u_sat = clip(u_raw, 0, 100)
    (untuk jalur kontrol, range 0-100% deviasi)
    │
    ├──────────────────────────────→ [Sum_Valve_Display + 65.69] → Scope_Valve
    │                                  (hanya untuk tampilan, ~65-70%)
    ▼
PENJUMLAHAN DISTURBANCE
    u_dist = u_sat + disturbance
    (gangguan ±3% ditambahkan ke sinyal valve)
    │
    ▼
DEAD TIME (θ = 1978s)
    u_delayed = u_dist pada t - 1978s
    (sinyal baru dirasakan proses setelah 33 menit)
    │
    ▼
TRANSFER FUNCTION FOPDT
    PV_deviasi = FOPDT(u_delayed) = 0.8616/(14471s+1) × u_delayed
    (respons proses dalam deviasi, nilai kecil ~0–1%)
    │
    ▼
PENJUMLAHAN OFFSET
    PV_absolut = PV_deviasi + 52.61
    (nilai PV yang sebenarnya, ~52–54%)
    │
    ├──────────────────────────────→ Scope (tampilkan PV)
    │
    └──────────────────────────────→ port (-) Sum (feedback)
                                        e = SP - PV_absolut (loop tertutup)
```

---

## 2.4 Parameter Simulasi

| Parameter | Nilai | Keterangan |
|---|---|---|
| Stop Time | 864000 s (10 hari) | Cukup untuk melihat step + disturbance |
| Solver | ode45 | Runge-Kutta orde 4-5 |
| Max Step | 60 s | Resolusi maksimum simulasi |

---

## 2.5 Hasil Simulasi Simulink

### Step Response (t=0 hingga t=43200s)

| | Existing PIDA | Tyreus-Luyben |
|---|---|---|
| Rise time | Lambat (~2 jam) | Lebih cepat (~40 menit) |
| Overshoot | Minimal | Sangat minimal |
| Steady state | ~53% | ~53% |

### Disturbance Rejection

| | Existing PIDA | Tyreus-Luyben |
|---|---|---|
| Range osilasi | ±0.5% | ±0.35% |
| Recovery speed | Lambat | Lebih cepat |
| Steady state | Kembali ke 53% | Kembali ke 53% |

### Sinyal Valve

| | Existing PIDA | Tyreus-Luyben |
|---|---|---|
| Steady state | ~65.5% | ~65.5% |
| Saat step | Naik gradual | Naik lebih agresif |
| Saat disturbance | Perubahan halus | Perubahan lebih besar |

---

## 2.6 Kesimpulan

### Google Colab → Simulink: Keterkaitan

| Colab | Simulink |
|---|---|
| Identifikasi Kp, τ, θ via PRBS | Dipakai di blok Transfer Fcn dan Transport Delay |
| Hitung Ku, Pu via frequency response | Dasar perhitungan parameter PID TL |
| Tuning TL: Kc, Ti, Td | Diinput ke blok PID Controller |
| Simulasi Python (validasi) | Diverifikasi ulang di Simulink |

### Tyreus-Luyben vs Existing PIDA

| Aspek | Existing PIDA | Tyreus-Luyben | Kesimpulan |
|---|---|---|---|
| **Kecepatan respons** | Lambat | 3× lebih cepat | TL lebih baik |
| **Akurasi steady state** | SS Error lebih besar | SS Error lebih kecil | TL lebih baik |
| **Stabilitas** | Stabil | Stabil dengan anti-windup | Sebanding |
| **Agresivitas valve** | Halus | Lebih agresif | Existing lebih ramah aktuator |
| **ISE/IAE/ITAE** | Lebih besar | Lebih kecil (~50-67%) | TL lebih baik |

### Catatan Implementasi

Tyreus-Luyben menunjukkan performa lebih baik di semua metrik error, namun memerlukan:
1. **Anti-windup** — wajib karena Kc lebih besar (rentan windup)
2. **Filter derivative** — N=5 atau lebih kecil untuk meredam noise
3. **Evaluasi aktuator** — gerakan valve lebih agresif perlu dipertimbangkan untuk umur valve

---

*Dokumentasi ini mencakup seluruh alur dari data historis DCS hingga validasi Simulink untuk sistem 34_LC_013 Level Steam Drum Controller, WHB_FEEDWATER, PT Pusri IB.*
