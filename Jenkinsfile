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
                    echo "=== Starting Snyk Scan ==="
                    sh '''
                        docker rm -f snyk-scan-job || true

                        # Create a script file to run inside container
                        cat > /tmp/snyk-scan.sh << 'SNYK_SCRIPT'
#!/bin/bash
cd /app
echo "=== Files in /app ==="
ls -la
echo "=== Running Snyk scan ==="
snyk test --all-projects --severity-threshold=high || true
echo "=== Generating JSON report ==="
snyk test --all-projects --json > /app/snyk-report.json || true
echo "=== Report generated ==="
ls -la /app/snyk-report.json
SNYK_SCRIPT

                        echo "Creating Snyk container..."
                        docker create --name snyk-scan-job \
                            -e SNYK_TOKEN=${SNYK_TOKEN} \
                            -w /app \
                            snyk/snyk:maven-3-jdk-17 \
                            bash /app/snyk-scan.sh

                        echo "Copying files to container..."
                        docker cp . snyk-scan-job:/app
                        docker cp /tmp/snyk-scan.sh snyk-scan-job:/app/snyk-scan.sh

                        echo "Running Snyk scan..."
                        docker start -a snyk-scan-job || true

                        echo "Copying report back..."
                        docker cp snyk-scan-job:/app/snyk-report.json . && echo "SUCCESS: Copied snyk-report.json" || echo "FAILED: Could not copy snyk-report.json"

                        ls -la snyk*.* 2>/dev/null || echo "No snyk files in workspace"
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
                    echo "=== Starting ZAP Scan ==="
                    sh '''
                        docker rm -f zap-scan-job || true

                        # Create script for ZAP that creates the wrk directory first
                        cat > /tmp/zap-scan.sh << 'ZAP_SCRIPT'
#!/bin/bash
mkdir -p /zap/wrk
echo "=== Starting ZAP baseline scan ==="
zap-baseline.py -t $1 -r zap-report.html -J zap-report.json -I
echo "=== Report generated ==="
ls -la /zap/wrk/
ZAP_SCRIPT

                        echo "Creating ZAP container..."
                        docker create --name zap-scan-job \
                            --user root \
                            zaproxy/zap-stable \
                            bash /tmp/zap-scan.sh "${APP_URL}"

                        echo "Copying script to container..."
                        docker cp /tmp/zap-scan.sh zap-scan-job:/tmp/zap-scan.sh

                        echo "Running ZAP scan..."
                        docker start -a zap-scan-job || true

                        echo "Copying reports back..."
                        docker cp zap-scan-job:/zap/wrk/zap-report.html . && echo "SUCCESS: Copied zap-report.html" || echo "FAILED: Could not copy zap-report.html"
                        docker cp zap-scan-job:/zap/wrk/zap-report.json . && echo "SUCCESS: Copied zap-report.json" || echo "FAILED: Could not copy zap-report.json"

                        ls -la zap-report.* 2>/dev/null || echo "No zap files in workspace"
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
