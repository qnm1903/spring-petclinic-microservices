# Hướng Dẫn Cài Đặt Pipeline DevSecOps

Tài liệu này hướng dẫn chi tiết các bước thiết lập môi trường DevSecOps cho dự án Spring Petclinic Microservices, bao gồm quy trình bảo mật tại máy trạm (Local Security) và tích hợp liên tục (CI Pipeline) với Jenkins.

## Mục lục

1.  [Thiết lập Môi trường Local (Pre-commit Hook)](#1-thiết-lập-môi-trường-local-pre-commit-hook)
2.  [Cài đặt & Cấu hình Jenkins](#2-cài-đặt--cấu-hình-jenkins)
3.  [Tích hợp Công cụ Bảo Mật (Sonar, Snyk, ZAP)](#3-tích-hợp-công-cụ-bảo-mật-sonar-snyk-zap)
4.  [Chạy Pipeline & Kiểm tra Kết quả](#4-chạy-pipeline--kiểm-tra-kết-quả)

---

## 1. Thiết lập Môi trường Local (Pre-commit Hook)

Mục đích: Ngăn chặn việc vô tình commit các thông tin nhạy cảm (API Key, Password) vào mã nguồn bằng công cụ **Gitleaks**.

### a. Cài đặt

Yêu cầu: Python đã được cài đặt.

```bash
# 1. Cài đặt framework pre-commit
pip install pre-commit

# 2. Di chuyển vào thư mục dự án
cd spring-petclinic-microservices

# 3. Cài đặt hooks vào git registry của dự án
pre-commit install
```

### b. Kiểm thử hoạt động

Để xác minh hook hoạt động, hãy thử tạo một "bí mật giả" và commit:

```bash
# Tạo file chứa secret giả
echo 'aws_secret_access_key = "AKIAJ7K7K7K7K7K7K7K7"' > demo/secret.env

# Thử commit (Mong đợi: Commit bị CHẶN)
git add demo/secret.env
git commit -m "test secret detection"
# Xem terminal: Gitleaks sẽ báo lỗi màu đỏ

# Dọn dẹp sau khi test
git reset HEAD demo/secret.env
rm demo/secret.env
```

---

## 2. Cài đặt & Cấu hình Jenkins

Mục đích: Dựng server Jenkins bằng Docker để chạy quy trình CI/CD.

### a. Khởi chạy Jenkins

Tại thư mục `jenkins/` của dự án, chạy lệnh:

```bash
docker-compose up -d
```
*   Dịch vụ sẽ chạy tại: **http://localhost:9080**

### b. Unlock Jenkins

Khi truy cập lần đầu, người dùng cần lấy mật khẩu Admin:

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```
*   Copy chuỗi ký tự trả về và dán vào màn hình setup.
*   Chọn **"Install suggested plugins"**.

### c. Cài đặt Plugins cần thiết

Vào **Manage Jenkins → Plugins → Available Plugins**, tìm và cài đặt:
1.  **SonarQube Scanner** (Bắt buộc)
2.  **HTML Publisher** (Để xem báo cáo ZAP)
3.  **Docker Pipeline** (Để chạy Snyk/ZAP trong container)
4.  **Maven Integration** (Hỗ trợ tốt hơn cho Java)
5.  **Pipeline: Stage View** (Để xem Pipeline theo từng stage) (Optional)

### d. Cấu hình Global Tools

Vào **Manage Jenkins → Tools**:

*   **JDK**:
    *   Name: `JDK-17`
    *   JAVA_HOME: `/opt/java/openjdk` (Đây là đường dẫn trong image `jenkins/jenkins:lts-jdk17`)
*   **Maven**:
    *   Name: `Maven-3.9`
    *   Check chọn **"Install automatically"**.

---

## 3. Tích hợp Công cụ Bảo Mật (Sonar, Snyk, ZAP)

### a. Chuẩn bị Tài Khoản & Token (Bên ngoài Jenkins)

1.  **SonarCloud**:
    *   Đăng nhập [sonarcloud.io](https://sonarcloud.io) bằng GitHub.
    *   Tạo Organization và Import dự án `spring-petclinic-microservices`.
    *   Vào **Account -> Security** để tạo Token.
2.  **Snyk**:
    *   Đăng ký tài khoản tại [snyk.io](https://snyk.io/).
    *   Vào **Account Settings** (góc trái dưới) -> **API Token** -> Lấy Key.

### b. Cấu hình Credentials trong Jenkins

Vào **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**:

| Kind | ID | Secret | Description |
| :--- | :--- | :--- | :--- |
| **Secret text** | `sonar-token` | *(Token từ SonarCloud)* | SonarCloud API Token |
| **Secret text** | `snyk-token` | *(Token từ Snyk)* | Snyk API Token |

### c. Kết nối SonarCloud với Jenkins

Vào **Manage Jenkins → System**:
*   Tìm mục **SonarQube servers**.
*   Check **Enable injection of SonarQube server configuration**.
*   **Name**: `SonarCloud` (Quan trọng: Phải khớp chính xác với tên trong `Jenkinsfile`).
*   **Server URL**: `https://sonarcloud.io`.
*   **Server authentication token**: Chọn credential `sonar-token` vừa tạo.

---

## 4. Chạy Pipeline & Kiểm tra Kết quả

### a. Tạo Job Pipeline

1.  Trên Jenkins Dashboard -> **New Item**.
2.  Đặt tên: `spring-petclinic-devsecops` -> Chọn **Pipeline**.
3.  Trong phần cấu hình **Pipeline**:
    *   **Definition**: `Pipeline script from SCM`.
    *   **SCM**: `Git`.
    *   **Repository URL**: Điền đường dẫn tới repo GitHub (hoặc đường dẫn local).
    *   **Script Path**: `Jenkinsfile`.
4.  **Save** và click **Build Now**.

### b. Xem Báo Cáo Bảo Mật

Sau khi Build thành công (Success), kiểm tra các artifact và link sau:

1.  **SonarQube Analysis**:
    *   Click vào icon SonarQube bên cạnh Build Number hoặc link trong Console Log.
    *   Kiểm tra: Bugs, Vulnerabilities, Coverage, Duplications.
2.  **Snyk Vulnerability Report**:
    *   Trong trang Job, tìm Artifact: `snyk-report.json`.
    *   File này chứa danh sách các CVE tìm thấy trong dependencies.
3.  **OWASP ZAP Report**:
    *   Trong trang Job, tìm link **OWASP ZAP Report** (nếu đã cài HTML Publisher).
    *   Hoặc tải Artifact `zap-report.html`.
    *   Chứa kết quả quét dynamic vulnerability trên ứng dụng.

---

## ⚠️ Khắc phục sự cố (Troubleshooting)

1.  **Lỗi "docker: not found" trong Jenkins**:
    *   Đảm bảo đã mount `docker.sock` khi chạy container Jenkins (xem file `jenkins/docker-compose.yml` hoặc lệnh run trong `jenkins/README.md`).
2.  **Lỗi "Quality Gate Failed"**:
    *   SonarCloud mặc định chặn build nếu Coverage < 80%. Cần vào SonarCloud Dashboard để điều chỉnh ngưỡng Quality Gate hoặc viết thêm Test.
3.  **Gitleaks báo "False Positive"**:
    *   Nếu Gitleaks chặn nhầm một chuỗi không phải secret, hãy thêm chuỗi đó vào file `.gitleaks.toml` phần `[allowlist]`.
