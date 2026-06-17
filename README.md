# 🔐 BooksLib - DevSecOps CI/CD Pipeline (Track A)

## 📌 Overview

Repositori ini merupakan implementasi **Secure SDLC (Software Development Lifecycle)** dengan mengintegrasikan keamanan di setiap fase pipeline CI/CD menggunakan Jenkins untuk aplikasi microservice BooksLib.
=======
# BooksLib - DevSecOps CI/CD Pipeline (Track A)

## Arsitektur & Workflow

Developer → GitHub (Git Flow) → Jenkins Pipeline

│

┌─────────────────┼─────────────────┐
 

│                 │                 │


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
=======
SAST Scan         Build Image        Image Scan

(Bandit/Gosec)    (docker compose)      (Trivy)


│                 │                 │


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
=======
└─────────────────┼─────────────────┘

│



(docker compose)

## Services
- **auth-service** - Go (port 8081)
- **books-service** - .NET 8 (port 8082)
- **reviews-service** - Python/Django (port 8083)
- **frontend** - React/Nginx (port 3000)
- **db** - PostgreSQL 15

## Git Flow Strategy
- `main` → production
- `develop` → integration
- `feature/*` → fitur baru
- `hotfix/*` → perbaikan urgent

## Pipeline Stages
1. **Checkout** - clone repo
2. **SAST - Bandit** - scan Python (reviews-service)
3. **SAST - Gosec** - scan Go (auth-service)
4. **Build** - build semua Docker image
5. **Image Scan - Trivy** - scan vulnerabilities di image
6. **Deploy** - deploy dengan docker-compose

## Cara Menjalankan

### Prerequisites
- Docker & Docker Compose
- Jenkins (via Docker)
>>>>>>> develop

### 1. Clone Repository
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
=======
git clone https://github.com/madesiregar/bookslib.git
cd bookslib
```

### 2. Setup Environment
```bash
cp .env.example .env
# Edit .env sesuai kebutuhan
```

### 3. Jalankan Aplikasi
```bash
docker compose up --build
```

Akses di:
- Frontend: http://localhost:3000
- Auth: http://localhost:8081
- Books: http://localhost:8082
- Reviews: http://localhost:8083

### 4. Jalankan Jenkins
```bash
docker run -d --name jenkins \
  -p 8090:8080 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts
```

## Security Findings (Trivy Scan)
Pipeline menemukan vulnerabilities pada:
- **auth-service**: 19 CVEs (2 CRITICAL, 17 HIGH) - OpenSSL, musl, Go stdlib outdated
- **reviews-service**: 30 CVEs (2 CRITICAL, 26 HIGH) - Django 4.2.7 outdated, Debian packages

## Trade-off & Keputusan Teknis
- **Jenkins** dipilih karena requirement wajib
- **Trivy** untuk image scanning karena gratis, cepat, dan comprehensive
- **Bandit + Gosec** untuk SAST karena native ke masing-masing bahasa
- `|| true` di scan stages agar pipeline tidak stop saat ada findings (report-only mode)


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
=======
## Kendala
- Port 8080 bentrok dengan XAMPP → Jenkins dipindah ke port 8090
- auth-service crash saat startup karena race condition dengan DB → fix dengan retry loop
- Gosec tidak bisa install karena versi Go 1.20 tidak kompatibel dengan gosec terbaru

## Yang Akan Diperbaiki Jika Ada Waktu Lebih
- Upgrade Go ke 1.21+ dan fix gosec
- Upgrade Django ke versi terbaru
- Implementasi zero-downtime deployment dengan K3s
- Tambah unit test stage di pipeline
- Implementasi secret scanning dengan GitLeaks
- Fix SQL injection di auth-service loginHandler

