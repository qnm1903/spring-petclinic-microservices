pipeline {
    agent any

    tools {
        maven 'Maven-3.9'  // Configure in Jenkins Global Tool Configuration
        jdk 'JDK-17'       // Configure in Jenkins Global Tool Configuration
    }

    environment {
        SONAR_TOKEN = credentials('sonar-token')           // SonarCloud token
        SNYK_TOKEN = credentials('snyk-token')             // Snyk token
        SONAR_HOST_URL = 'https://sonarcloud.io'
        SONAR_ORG = 'qnm1903'                              // Replace with your org
        APP_URL = 'http://host.docker.internal:8080'        // App URL for ZAP (from inside container)
        RUN_ZAP = 'true'                                    // Enable ZAP scan
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo 'Code checked out successfully'
            }
        }

        stage('Build') {
            steps {
                sh './mvnw clean package -DskipTests'
            }
            post {
                success {
                    echo 'Build completed successfully'
                }
            }
        }

        stage('Unit Tests') {
            steps {
                sh './mvnw test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarCloud Analysis') {
            steps {
                withSonarQubeEnv('SonarCloud') {
                    sh '''
                        ./mvnw org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                            -Dsonar.projectKey=qnm1903_spring-petclinic-microservices \
                            -Dsonar.organization=${SONAR_ORG} \
                            -Dsonar.host.url=${SONAR_HOST_URL} \
                            -Dsonar.token=${SONAR_TOKEN}
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Snyk Security Scan') {
            steps {
                script {
                    echo "=== Starting Snyk Scan (Copy Mode) ==="
                    sh '''
                        # Remove old container if exists
                        docker rm -f snyk-scan-job || true

                        echo "Creating Snyk container..."
                        docker create --name snyk-scan-job \
                            -e SNYK_TOKEN=${SNYK_TOKEN} \
                            -w /app \
                            snyk/snyk:maven-3-jdk-17 \
                            sh -c "ls -la /app && snyk test --all-projects --severity-threshold=high; snyk test --all-projects --json > /app/snyk-report.json; ls -la /app"

                        echo "Copying project files to Snyk container..."
                        docker cp . snyk-scan-job:/app

                        echo "Running Snyk scan..."
                        docker start -a snyk-scan-job || true

                        echo "=== DEBUG: Listing files in container /app ==="
                        docker start snyk-scan-job 2>/dev/null || true
                        docker exec snyk-scan-job ls -la /app 2>/dev/null || docker cp snyk-scan-job:/app/. /tmp/snyk-debug 2>/dev/null || echo "Cannot list container files"

                        echo "Copying report back..."
                        docker cp snyk-scan-job:/app/snyk-report.json . && echo "SUCCESS: Copied snyk-report.json" || echo "FAILED: Could not copy snyk-report.json"

                        echo "=== Files in workspace ==="
                        ls -la snyk*.* 2>/dev/null || echo "No snyk files found"

                        # Cleanup
                        docker rm -f snyk-scan-job || true
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'snyk-report.json', allowEmptyArchive: true
                }
            }
        }

        stage('OWASP ZAP Scan') {
            when {
                expression {
                    return env.RUN_ZAP == 'true'
                }
            }
            steps {
                script {
                    echo "=== Starting ZAP Scan (Copy Mode) ==="
                    sh '''
                        docker rm -f zap-scan-job || true

                        echo "Creating ZAP container..."
                        docker create --name zap-scan-job \
                            --user root \
                            zaproxy/zap-stable \
                            sh -c "zap-baseline.py -t ${APP_URL} -r /zap/wrk/zap-report.html -J /zap/wrk/zap-report.json -I; ls -la /zap/wrk/"

                        echo "Running ZAP scan..."
                        docker start -a zap-scan-job || true

                        echo "=== DEBUG: Listing files in container ==="
                        docker diff zap-scan-job 2>/dev/null | head -20 || echo "Cannot diff container"

                        echo "Trying multiple paths to copy reports..."
                        docker cp zap-scan-job:/zap/wrk/zap-report.html . 2>/dev/null && echo "Copied from /zap/wrk/" || \
                        docker cp zap-scan-job:/home/zap/zap-report.html . 2>/dev/null && echo "Copied from /home/zap/" || \
                        docker cp zap-scan-job:/zap/zap-report.html . 2>/dev/null && echo "Copied from /zap/" || \
                        echo "FAILED: Could not find zap-report.html"

                        docker cp zap-scan-job:/zap/wrk/zap-report.json . 2>/dev/null || \
                        docker cp zap-scan-job:/home/zap/zap-report.json . 2>/dev/null || \
                        docker cp zap-scan-job:/zap/zap-report.json . 2>/dev/null || \
                        echo "FAILED: Could not find zap-report.json"

                        echo "=== Files in workspace ==="
                        ls -la zap-report.* *.html *.json 2>/dev/null || echo "No report files found"

                        # Cleanup
                        docker rm -f zap-scan-job || true
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'zap-report.*', allowEmptyArchive: true
                    publishHTML(target: [
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: '.',
                        reportFiles: 'zap-report.html',
                        reportName: 'OWASP ZAP Report'
                    ])
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
