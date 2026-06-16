# 🔐 BooksLib - DevSecOps CI/CD Pipeline (Track A)

## 📌 Overview

Repositori ini merupakan implementasi **Secure SDLC (Software Development Lifecycle)** dengan mengintegrasikan keamanan di setiap fase pipeline CI/CD menggunakan Jenkins untuk aplikasi microservice BooksLib.

---

## 🏗️ Arsitektur & Alur Kerja (Workflow)

### Gambaran Sistem

| Layer | Komponen | Keterangan |
|---|---|---|
| Source Control | GitHub + Git Flow | Branch management & versioning |
| CI/CD Engine | Jenkins | Otomatisasi pipeline |
| SAST | Bandit + Gosec | Static Application Security Testing |
| Build | Docker Compose | Build semua Docker image |
| Image Security | Trivy | Container vulnerability scanning |
| Deployment | Docker Compose | Deploy semua service |

### Alur Pipeline

**Developer push code → GitHub → Jenkins otomatis:**

| # | Stage | Tool | Tujuan Keamanan |
|---|---|---|---|
| 1 | Checkout | Git | Ambil source code terbaru |
| 2 | SAST Python | Bandit | Deteksi celah keamanan di reviews-service |
| 3 | SAST Go | Gosec | Deteksi celah keamanan di auth-service |
| 4 | Build Image | Docker Compose | Build semua service menjadi Docker image |
| 5 | Image Scan | Trivy | Scan vulnerabilities di image sebelum deploy |
| 6 | Deploy | Docker Compose | Deploy hanya jika semua tahap sebelumnya aman |

### Git Flow Strategy

| Branch | Fungsi |
|---|---|
| `main` | Production — kode yang sudah stabil |
| `develop` | Integration — merge semua fitur |
| `feature/*` | Pengembangan fitur baru |
| `hotfix/*` | Perbaikan bug urgent di production |

### Stack Aplikasi

| Service | Tech Stack | Port |
|---|---|---|
| auth-service | Go 1.20 | 8081 |
| books-service | .NET 8 | 8082 |
| reviews-service | Python/Django 4.2 | 8083 |
| frontend | React + Nginx | 3000 |
| db | PostgreSQL 15 | 5432 |

---

## 🚀 Langkah Reproduksi (Running/Deploying)

### Prerequisites
- Docker & Docker Compose
- Jenkins (running sebagai Docker container)
- Git

### 1. Fork & Clone Repository
```bash
git clone https://github.com/madesiregar/bookslib.git
cd bookslib
```

### 2. Setup Environment Variables
```bash
cp .env.example .env
nano .env
# Isi dengan:
# POSTGRES_USER=postgres
# POSTGRES_PASSWORD=yourpassword
# POSTGRES_DB=postgres
# DB_HOST=db
```

### 3. Jalankan Aplikasi Manual
```bash
docker compose up --build
```

| Service | URL |
|---|---|
| Frontend | http://localhost:3000 |
| Auth API | http://localhost:8081 |
| Books API | http://localhost:8082 |
| Reviews API | http://localhost:8083 |

### 4. Setup Jenkins

```bash
mkdir -p ~/jenkins
cd ~/jenkins
# Buat docker-compose.yml untuk Jenkins
docker compose up -d
```

Buka Jenkins di: **http://localhost:8090**

### 5. Setup Jenkins Pipeline

1. Buka Jenkins → New Item → Pipeline
2. Name: `bookslib-pipeline`
3. Pipeline → Pipeline script from SCM
4. SCM: Git → URL: `https://github.com/madesiregar/bookslib.git`
5. Branch: `*/develop`
6. Script Path: `Jenkinsfile`
7. Add Credentials: `.env` file sebagai Secret File dengan ID `bookslib-env`
8. Save → Build Now

### 6. Verifikasi Pipeline

Pipeline akan otomatis menjalankan:
- ✅ SAST scan (Bandit + Gosec)
- ✅ Build Docker images
- ✅ Trivy vulnerability scan
- ✅ Deploy dengan docker-compose

---

## 🔍 Security Findings

### Temuan dari Trivy Image Scan

| Service | Total CVE | Critical | High | Sumber |
|---|---|---|---|---|
| auth-service | 19 | 2 | 17 | OpenSSL, musl, Go stdlib 1.20 |
| reviews-service | 30 | 2 | 28 | Django 4.2.7, Debian packages |

### Temuan dari Code Review (Didokumentasikan sebagai GitHub Issues)

| Issue | Severity | Status |
|---|---|---|
| SQL Injection di loginHandler | Critical | Open |
| Password stored in plaintext | High | Open |
| Hardcoded credentials in source code | High | Open |
| auth-service crash on startup | Medium | Fixed ✅ |

---

## ⚖️ Trade-off & Alasan Pemilihan Tools

| Tools | Alasan Dipilih | Trade-off |
|---|---|---|
| **Jenkins** | Requirement wajib, self-hosted, full control | Setup lebih kompleks vs GitHub Actions |
| **Trivy** | Gratis, cepat, support Docker image + filesystem scan | Perlu download DB saat pertama run |
| **Bandit** | Native untuk Python, mudah diintegrasikan | Hanya untuk Python |
| **Gosec** | Native untuk Go, deteksi security pattern | Tidak kompatibel dengan Go 1.20 terbaru |
| **Docker Compose** | Simple, sesuai requirement minimal | Tidak support zero-downtime out of the box |
| **`\|\| true`** di scan | Pipeline tidak stop saat ada findings (report-only) | Findings tidak memblock deployment |

---

## 🚧 Kendala yang Dihadapi

| Kendala | Solusi |
|---|---|
| Port 8080 bentrok dengan XAMPP | Jenkins dipindah ke port 8090 |
| auth-service crash saat startup (race condition dengan DB) | Tambah retry loop 10x dengan interval 3 detik |
| Gosec tidak kompatibel dengan Go 1.20 | Skip dengan `\|\| true`, dicatat sebagai kendala |
| `.env` ter-push ke GitHub | Langsung di-remove dari git tracking |
| Frontend tidak bisa connect ke backend | Fix environment variable VITE_* ke IP yang benar |

---

## 🔮 Yang Akan Dilakukan Jika Ada Waktu Lebih

- [ ] **Upgrade Go 1.20 → 1.21+** dan fix Gosec compatibility
- [ ] **Upgrade Django 4.2.7 → latest** untuk fix SQL injection CVEs
- [ ] **Fix SQL Injection** di `auth-service/loginHandler` dengan parameterized query
- [ ] **Implementasi bcrypt** untuk password hashing
- [ ] **Hapus hardcoded credentials** dari source code
- [ ] **Implementasi zero-downtime deployment** dengan K3s/K8s
- [ ] **Tambah unit test stage** di pipeline
- [ ] **Implementasi secret scanning** dengan GitLeaks
- [ ] **Setup webhook GitHub → Jenkins** untuk trigger otomatis saat push
