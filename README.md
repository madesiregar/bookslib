# BooksLib - DevSecOps CI/CD Pipeline (Track A)

## Arsitektur & Workflow

Developer → GitHub (Git Flow) → Jenkins Pipeline

│

┌─────────────────┼─────────────────┐

│                 │                 │

SAST Scan         Build Image        Image Scan

(Bandit/Gosec)    (docker compose)      (Trivy)

│                 │                 │

└─────────────────┼─────────────────┘

│

Deploy

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

### 1. Clone Repository
```bash
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
