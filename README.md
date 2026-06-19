# 📚 BooksLib - DevSecOps CI/CD Pipeline (Track A)

Repositori ini berisi implementasi **Secure SDLC (Software Development Lifecycle)** dan pipeline **DevSecOps Automation** berbasis **Jenkins** untuk aplikasi *BooksLib*. Sistem ini dirancang untuk mendeteksi celah keamanan sedini mungkin (*Shift-Left Security*) sebelum aplikasi otomatis dideploy menggunakan Docker Compose.

---

## 🏗️ 1. Arsitektur & Alur Kerja (Workflow) Pipeline

Pipeline ini menerapkan strategi **Git Flow** (fokus pada branch `develop` untuk staging dan `main`/`master` untuk production). Setiap kali developer melakukan `git push`, Jenkins akan memicu rangkaian tahap berikut:

```
[ Code Push ] ──> [ Checkout SCM ]
                        │
                        ▼
            [ Stage: Security Scanning ]
            ├── Gitleaks (Secret Scanning)
            ├── Semgrep (SAST)
            └── Dependency Scan (Python/Node/Go)
                        │
                        ▼
            [ Stage: Build Docker Images ]
                        │
                        ▼
            [ Stage: Container Scan (Trivy) ] ──> [ Automated GitHub Issues Alert ]
                        │
                        ▼
            [ Stage: Deploy to Staging ] (Docker Compose)
```

### Detail Tahapan Pipeline:

1. **Checkout SCM**: Mengambil kode sumber terbaru dari GitHub.
2. **Secret Scanning (Gitleaks)**: Memindai riwayat komit untuk memastikan tidak ada hardcoded credentials, token, atau private key yang bocor.
3. **SAST (Semgrep)**: Melakukan analisis kode statis untuk mencari pola kode yang tidak aman atau rentan terhadap serangan (seperti SQL Injection, XSS, dll).
4. **Dependency Scan**: Memeriksa kerentanan pada pustaka pihak ketiga menggunakan `pip-audit`, `npm audit`, dan `govulncheck`.
5. **Build Docker Images**: Melakukan kompilasi aplikasi microservices menjadi container image lokal melalui Docker Compose Build.
6. **Image Scanning (Trivy)**: Memindai lapisan base image OS dan dependensi di dalam container untuk mencari kerentanan (CVE) dengan tingkat keparahan High dan Critical.
7. **Document Automation to GitHub Issues**: Jika pipeline mendeteksi adanya aktivitas pemindaian, sistem akan otomatis membuka Issue baru di GitHub sebagai alert bagi tim developer.
8. **Automated Deployment**: Jika seluruh tahap aman, aplikasi dideploy secara otomatis ke lingkungan Staging menggunakan Docker Compose.

---

## 🚀 2. Langkah-Langkah Mereproduksi Sistem (Running/Deploying)

### Prasyarat (Prerequisites)

Pastikan server/VM Anda sudah terpasang OS Linux (Ubuntu/Debian) dengan komponen berikut:

- Docker & Docker Compose v2
- Jenkins (Self-hosted atau Docker-based)
- Akun GitHub dan Personal Access Token (PAT) dengan hak akses repo (untuk menulis GitHub Issues)

### Langkah 1: Konfigurasi Kredensial di Jenkins

1. Masuk ke Dashboard Jenkins Anda.
2. Pergi ke **Manage Jenkins > Credentials > System > Global credentials (unrestricted)**.
3. Tambahkan kredensial berikut:
   - **ID**: `github-credentials` — **Jenis**: Username with password (Isi username GitHub Anda dan Token PAT sebagai password).

### Langkah 2: Membuat Pipeline di Jenkins

1. Klik **New Item**, masukkan nama `bookslib-pipeline`, pilih **Pipeline**, lalu klik **OK**.
2. Pada bagian **Build Triggers**, centang **GitHub hook trigger for GITScm polling** agar build berjalan otomatis setiap kali ada push.
3. Pada bagian **Pipeline**, ubah **Definition** menjadi **Pipeline script from SCM**.
4. Pilih **Git**, masukkan URL repositori Anda (misal: `https://github.com/username/bookslib.git`).
5. Ubah **Branch Specifier** menjadi `*/develop` (sesuai Git Flow).
6. Pastikan **Script Path** mengarah ke `Jenkinsfile`. Klik **Save**.

### Langkah 3: Menjalankan Pipeline

Lakukan perubahan kode di lokal, lalu jalankan perintah:

```
git checkout develop
git add .
git commit -m "feat: trigger devsecops pipeline"
git push origin develop
```

Jenkins akan otomatis memulai proses build. Anda dapat melihat log jalannya pipeline pada menu **Console Output** di Jenkins.

---

## ⚖️ 3. Trade-off & Alasan Pemilihan Tools / Strategi

Dalam merancang arsitektur ini, beberapa keputusan teknologi diambil berdasarkan pertimbangan efisiensi dan kebutuhan industri:

- **Jenkins (Mandatori)**: Dipilih karena fleksibilitasnya yang luar biasa tinggi melalui konsep Pipeline-as-Code (Jenkinsfile) dan sifatnya yang self-hosted, sehingga seluruh data kode dan laporan keamanan tidak keluar ke pihak ketiga.
- **Gitleaks & Semgrep via Docker**: Menjalankan tooling keamanan di dalam container Docker memastikan lingkungan pemindaian yang konsisten tanpa perlu menginstal dependensi bahasa pemrograman secara lokal di server Jenkins.
- **Trivy vs Scanner Lain**: Trivy dipilih karena eksekusinya yang sangat cepat, database kerentanan yang selalu diperbarui, dan kemampuannya memindai image multi-bahasa sekaligus.
- **Docker Compose vs Kubernetes (untuk Staging)**: Untuk lingkungan Staging awal, menggunakan Docker Compose jauh lebih ringan, hemat sumber daya (RAM/CPU), dan mudah dikonfigurasi dibandingkan harus memelihara klaster Kubernetes yang kompleks sejak awal.

---

## 🛑 4. Kendala yang Dihadapi & Rencana Masa Depan

### Kendala yang Dihadapi selama Pengembangan:

- **Konteks Variabel Lingkungan Jenkins**: Terdapat isu di mana Jenkins mendeteksi variabel `env.GIT_BRANCH` dengan awalan `origin/develop`. Hal ini menyebabkan kegagalan sistem saat melakukan penamaan atau pelaporan teks ke luar. Masalah ini diselesaikan dengan membuat logika pembersihan string (`CLEAN_BRANCH`) memanfaatkan fungsi regex Groovy.
- **Keterbatasan Struktur Kode Microservices**: Beberapa sub-layanan belum menginisialisasi modul dependensi secara standar (misalnya absennya file `go.mod` di direktori tertentu atau kesalahan path `requirements.txt`), yang menyebabkan alat pemindai dependensi seperti `govulncheck` memunculkan peringatan (warning/error exit). Solusinya, perintah tersebut sementara di-bypass menggunakan operator `|| true` agar pipeline utama tidak putus selagi restrukturisasi kode berjalan.

### Apa yang Akan Dilakukan Jika Memiliki Waktu Lebih Banyak (Future Improvements):

- **Implementasi Strategi Zero Downtime (Nilai Plus)**: Bermigrasi dari Docker Compose ke Kubernetes (K3s) untuk memanfaatkan strategi Rolling Update / Rollout Deployment. Dengan begitu, saat aplikasi `auth` atau `books` diperbarui, container lama tidak akan mati sebelum container baru berstatus Healthy, menjamin Zero Downtime.
- **Logika Alerting yang Cerdas**: Saat ini alert ke GitHub Issues dikirimkan pada setiap proses pemindaian berjalan. Ke depannya, payload curl ke GitHub API akan dikombinasikan dengan kondisi Conditional (misal menggunakan tool `jq`), sehingga Issue hanya akan dibuat jika total temuan kerentanan > 0.
- **Integrasi DAST**: Menambahkan tahap pemindaian dinamis (Dynamic Application Security Testing) setelah aplikasi berhasil dideploy ke staging menggunakan OWASP ZAP untuk menguji celah keamanan pada aplikasi yang sedang berjalan.