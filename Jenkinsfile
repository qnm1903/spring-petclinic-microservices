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
                    # Use Snyk CLI via Docker - scan Maven projects
                    docker run --rm \
                        -e SNYK_TOKEN=${SNYK_TOKEN} \
                        -v $(pwd):/app \
                        -w /app \
                        snyk/snyk-cli:maven-3.9.6_java-17 \
                        snyk test --all-projects --severity-threshold=high || true

                    # Generate JSON report
                    docker run --rm \
                        -e SNYK_TOKEN=${SNYK_TOKEN} \
                        -v $(pwd):/app \
                        -w /app \
                        snyk/snyk-cli:maven-3.9.6_java-17 \
                        sh -c "snyk test --all-projects --json > snyk-report.json" || true
                '''
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
                sh '''
                    echo "Scanning target: ${APP_URL}"
                    # Create reports directory with proper permissions
                    mkdir -p zap-reports
                    chmod 777 zap-reports

                    docker run --rm \
                        -u root \
                        -v $(pwd)/zap-reports:/zap/wrk:rw \
                        zaproxy/zap-stable zap-baseline.py \
                        -t ${APP_URL} \
                        -r zap-report.html \
                        -J zap-report.json \
                        -I || true

                    # Move reports to workspace root
                    cp zap-reports/zap-report.* . 2>/dev/null || true
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
