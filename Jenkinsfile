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
                sh '''
                    echo "=== Starting Snyk Scan ==="
                    echo "Current directory: $(pwd)"
                    ls -la pom.xml || echo "No pom.xml in root"

                    # Run Snyk test and capture output
                    docker run --rm \
                        -e SNYK_TOKEN=${SNYK_TOKEN} \
                        -v "$(pwd)":/app \
                        -w /app \
                        snyk/snyk:maven-3-jdk-17 \
                        snyk test --all-projects --severity-threshold=high 2>&1 | tee snyk-output.txt || true

                    # Create JSON report - run snyk with json output inside container and redirect
                    docker run --rm \
                        -e SNYK_TOKEN=${SNYK_TOKEN} \
                        -v "$(pwd)":/app \
                        -w /app \
                        snyk/snyk:maven-3-jdk-17 \
                        sh -c "snyk test --all-projects --json 2>&1" > snyk-report.json || true

                    echo "=== Snyk files created ==="
                    ls -la snyk*.* 2>/dev/null || echo "No snyk files found"
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'snyk-*.json,snyk-*.txt', allowEmptyArchive: true
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
                sh '''
                    echo "=== Starting ZAP Scan ==="
                    echo "Scanning target: ${APP_URL}"

                    # Use explicit absolute path for mount
                    WORKSPACE_DIR=$(pwd)
                    echo "Workspace: ${WORKSPACE_DIR}"

                    docker run --rm \
                        --user root \
                        -v "${WORKSPACE_DIR}":/zap/wrk:rw \
                        zaproxy/zap-stable zap-baseline.py \
                        -t ${APP_URL} \
                        -r zap-report.html \
                        -J zap-report.json \
                        -I || true

                    echo "=== ZAP files created ==="
                    ls -la zap-report.* 2>/dev/null || echo "No zap-report files in current dir"
                    ls -la *.html *.json 2>/dev/null || echo "No report files found"
                '''
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
