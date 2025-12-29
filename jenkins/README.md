# Jenkins Setup Guide

## 1. Start Jenkins

Chạy lệnh sau tại thư mục này:

```bash
docker-compose up -d
```

Jenkins sẽ chạy tại: **http://localhost:8081** (để tránh xung đột với ứng dụng chạy port 8080).

## 2. Unlock Jenkins

Lấy mật khẩu admin ban đầu:

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Copy mật khẩu này và dán vào màn hình Setup của Jenkins.

## 3. Install Plugins

Chọn **"Install suggested plugins"**. Sau đó cài thêm các plugins sau (Manage Jenkins → Plugins → Available):

1.  **SonarQube Scanner** (cho SonarCloud)
2.  **Pipeline: Stage View** (thường đã có sẵn)
3.  **Docker Pipeline** (để chạy agent docker nếu cần)
4.  **HTML Publisher** (để xem report của ZAP)
5.  **Maven Integration** (nếu cần job maven cũ, pipeline thì không bắt buộc nhưng nên có)

## 4. Global Tool Configuration

Vào **Manage Jenkins → Tools**:

1.  **Maven**:
    *   Name: `Maven-3.9`
    *   Check "Install automatically" (hoặc trỏ tới maven local)
2.  **JDK**:
    *   Image `jenkins/jenkins:lts-jdk17` đã có sẵn JDK 17.
    *   Name: `JDK-17`
    *   JAVA_HOME: `/opt/java/openjdk` (Đây là đường dẫn mặc định trong image này)

## 5. Configure Credentials

Vào **Manage Jenkins → Credentials → System → Global credentials**:

### a. SonarCloud Token
*   **Kind**: Secret text
*   **Secret**: (Mã token bạn lấy từ SonarCloud)
*   **ID**: `sonar-token`
*   **Description**: SonarCloud Token

### b. Snyk Token
*   **Kind**: Secret text
*   **Secret**: (Mã token lấy từ account setting của Snyk)
*   **ID**: `snyk-token`
*   **Description**: Snyk API Token

## 6. Configure System (SonarQube)

Vào **Manage Jenkins → System**:
*   Tìm mục **SonarQube servers**
*   Check **Enable injection of SonarQube server configuration...**
*   **Name**: `SonarCloud` (tên này phải khớp với `withSonarQubeEnv('SonarCloud')` trong Jenkinsfile)
*   **Server URL**: `https://sonarcloud.io`
*   **Server authentication token**: Chọn credential `SonarCloud Token` vừa tạo.

---
## 7. Tạo Pipeline Job

1.  **New Item** → Chọn **Pipeline** → Đặt tên `spring-petclinic-devsecops`.
2.  Trong phần **Pipeline**:
    *   Definition: **Pipeline script from SCM**
    *   SCM: **Git**
    *   Repository URL: (URL local folder hoặc github repo của bạn)
    *   Script Path: `Jenkinsfile`
3.  **Save** và **Build Now**.
