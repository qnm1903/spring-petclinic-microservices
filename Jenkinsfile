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
                    // 1. Create container (stopped)
                    // We use sh -c to run multiple commands: text scan (to stdout) AND json scan (to file)
                    def snykCmd = "snyk test --all-projects --severity-threshold=high; snyk test --all-projects --json > snyk-report.json"

                    // Note: We avoid 'docker run' with volumes. We use 'docker create'.
                    sh '''
                        # Remove old container if exists
                        docker rm -f snyk-scan-job || true

                        echo "Creating Snyk container..."
                        docker create --name snyk-scan-job \
                            -e SNYK_TOKEN=${SNYK_TOKEN} \
                            -w /app \
                            snyk/snyk:maven-3-jdk-17 \
                            sh -c "snyk test --all-projects --severity-threshold=high || true; snyk test --all-projects --json > snyk-report.json || true"

                        echo "Copying project files to Snyk container..."
                        # Copy current directory content to /app in container
                        docker cp . snyk-scan-job:/app

                        echo "Running Snyk scan..."
                        docker start -a snyk-scan-job || true

                        echo "Copying report back..."
                        docker cp snyk-scan-job:/app/snyk-report.json . || echo "Failed to copy Snyk report"

                        # Cleanup
                        docker rm -f snyk-scan-job
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

                        # ZAP writes to /zap/wrk by default with zap-baseline.py
                        # We don't need to copy source code IN, just reports OUT.
                        # But we must verify if zap-baseline.py needs a mapped volume to write.
                        # Usually it writes to mounted volume. If not mounted, it writes to container path.

                        echo "Creating ZAP container..."
                        docker create --name zap-scan-job \
                            --user root \
                            zaproxy/zap-stable \
                            zap-baseline.py \
                            -t ${APP_URL} \
                            -r zap-report.html \
                            -J zap-report.json \
                            -I

                        echo "Running ZAP scan..."
                        docker start -a zap-scan-job || true

                        echo "Copying reports back..."
                        # Default workdir of zaproxy is /zap. py script mounts to /zap/wrk
                        # Without mount, files should represent in /zap/wrk inside container if the script creates it
                        # Or strictly follow the -r path.
                        # Let's try copying from /zap/wrk/

                        docker cp zap-scan-job:/zap/wrk/zap-report.html . || echo "Failed to copy HTML report"
                        docker cp zap-scan-job:/zap/wrk/zap-report.json . || echo "Failed to copy JSON report"

                        docker rm -f zap-scan-job
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
