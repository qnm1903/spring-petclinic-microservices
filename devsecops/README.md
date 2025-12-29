# DevSecOps Implementation Guide

Hướng dẫn triển khai DevSecOps cho Spring Petclinic Microservices.

## 📋 Mục lục

1. [Gitleaks Pre-commit Hook](#1-gitleaks-pre-commit-hook)
2. [SonarCloud (SAST)](#2-sonarcloud-sast)
3. [Snyk (Dependency Scan)](#3-snyk-dependency-scan)
4. [OWASP ZAP (DAST)](#4-owasp-zap-dast)
5. [Jenkins Pipeline](#5-jenkins-pipeline)

---

## 1. Gitleaks Pre-commit Hook

### Mục đích
Phát hiện secrets (API keys, passwords, tokens) trước khi commit vào repository.

### Cài đặt

```bash
# 1. Cài đặt pre-commit framework
pip install pre-commit

# 2. Cài đặt hooks vào repository
cd spring-petclinic-microservices
pre-commit install

# 3. (Optional) Chạy scan trên toàn bộ repo
pre-commit run --all-files
```

### Kiểm tra hoạt động

```bash
# Tạo file test với secret mẫu
echo 'aws_secret_access_key = "AKIAIOSFODNN7EXAMPLE"' > test-secret.txt

# Thử commit (sẽ bị chặn)
git add test-secret.txt
git commit -m "test secret detection"

# Kết quả mong đợi: Commit bị chặn với thông báo từ gitleaks

# Dọn dẹp
git reset HEAD test-secret.txt
rm test-secret.txt
```

### Cấu hình

| File | Mô tả |
|------|-------|
| `.pre-commit-config.yaml` | Cấu hình hooks (gitleaks, trailing-whitespace, etc.) |
| `.gitleaks.toml` | Custom rules và allowlist cho gitleaks |

### Bypass (khi cần thiết)

```bash
# Bypass tất cả hooks
git commit --no-verify -m "message"

# Bypass chỉ gitleaks
SKIP=gitleaks git commit -m "message"
```

---

## 2. SonarCloud (SAST)

### Mục đích
Phân tích static code để phát hiện bugs, vulnerabilities, và code smells.

### Setup SonarCloud

1. **Đăng nhập SonarCloud**
   - Truy cập [sonarcloud.io](https://sonarcloud.io)
   - Đăng nhập bằng GitHub account

2. **Import Repository**
   - Click "+" → "Analyze new project"
   - Chọn `spring-petclinic-microservices`
   - Tạo organization (nếu chưa có)

3. **Lấy Token**
   - Account → Security → Generate Token
   - Lưu token để dùng trong Jenkins

4. **Tắt Automatic Analysis**
   - Administration → Analysis Method
   - Tắt "Automatic Analysis" (vì dùng CI-based)

### Cấu hình

| File | Mô tả |
|------|-------|
| `sonar-project.properties` | Cấu hình project SonarCloud |
| `Jenkinsfile` | Pipeline với SonarCloud stage |

### Chạy Local (Optional)

```bash
# Cần có SONAR_TOKEN environment variable
./mvnw sonar:sonar \
    -Dsonar.projectKey=spring-petclinic-microservices \
    -Dsonar.organization=YOUR_ORG \
    -Dsonar.host.url=https://sonarcloud.io \
    -Dsonar.token=$SONAR_TOKEN
```

### Xem Results
- Dashboard: `https://sonarcloud.io/project/overview?id=spring-petclinic-microservices`

---

## 3. Snyk (Dependency Scan)

### Mục đích
Quét dependencies để phát hiện vulnerabilities và license issues.

### Setup

```bash
# Cài đặt Snyk CLI
npm install -g snyk

# Đăng nhập
snyk auth

# Quét project
snyk test --all-projects
```

### Jenkins Integration
Đã được tích hợp trong `Jenkinsfile` stage "Snyk Security Scan".

---

## 4. OWASP ZAP (DAST)

### Mục đích
Dynamic Application Security Testing - quét ứng dụng đang chạy để phát hiện vulnerabilities.

### Yêu cầu
- Docker installed
- Ứng dụng đang chạy tại `http://localhost:8080`

### Chạy Local

```bash
# Baseline scan (passive)
docker run --rm --network host \
    -v $(pwd):/zap/wrk/:rw \
    zaproxy/zap-stable zap-baseline.py \
    -t http://localhost:8080 \
    -r zap-report.html \
    -J zap-report.json

# Active scan (more thorough but intrusive)
docker run --rm --network host \
    -v $(pwd):/zap/wrk/:rw \
    zaproxy/zap-stable zap-full-scan.py \
    -t http://localhost:8080 \
    -r zap-full-report.html
```

---

## 5. Jenkins Pipeline

### Cài đặt Jenkins (Docker)

```bash
# Tạo volume
docker volume create jenkins_home

# Chạy Jenkins
docker run -d --name jenkins \
    -p 8081:8080 -p 50000:50000 \
    -v jenkins_home:/var/jenkins_home \
    -v /var/run/docker.sock:/var/run/docker.sock \
    jenkins/jenkins:lts

# Lấy initial password
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

### Required Plugins
- SonarQube Scanner
- Pipeline
- Git
- HTML Publisher
- JUnit

### Credentials Setup
Trong Jenkins → Manage Jenkins → Credentials:
- `sonar-token`: SonarCloud token
- `snyk-token`: Snyk API token

---

## Troubleshooting

### Pre-commit không hoạt động
```bash
# Kiểm tra cài đặt
pre-commit --version

# Cài lại hooks
pre-commit install --force
```

### Gitleaks báo false positive
Thêm pattern vào `.gitleaks.toml`:
```toml
[allowlist]
regexes = [
    '''your-false-positive-pattern''',
]
```
