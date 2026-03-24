pipeline {
    agent any

    environment {
        JAVA_HOME     = tool 'jdk17'
        SCANNER_HOME  = tool 'sonar-scanner'
        PATH = "${JAVA_HOME}/bin:/usr/bin:/bin"
        AWS_REGION = 'ap-south-1'
        ECR_REPO   = '695418678781.dkr.ecr.ap-south-1.amazonaws.com/medyaan-devices'
        IMAGE_TAG  = "v1.0.${BUILD_NUMBER}"
        DEPLOY_DIR = '/opt/Medyaan_Dev/medyaan_backend'
        DC_DATA_DIR = '/var/lib/jenkins/.dependency-check'
    }

    parameters {
        booleanParam(name: 'RUN_TEST', defaultValue: true)
        booleanParam(name: 'RUN_SONAR', defaultValue: true)
        booleanParam(name: 'RUN_QUALITY_GATE', defaultValue: true)
        booleanParam(name: 'RUN_OWASP', defaultValue: true)
        booleanParam(name: 'RUN_TRIVY_IMAGE', defaultValue: true)
        booleanParam(name: 'RUN_TRIVY_FS', defaultValue: true)
        booleanParam(name: 'RUN_DEPLOY', defaultValue: true)
    }

    stages {

        stage('Clean Workspace') {
            steps { cleanWs() }
        }

        stage('Checkout from CodeCommit') {
            steps {
                git branch: 'master',
                    credentialsId: 'DatayaanCICD',
                    url: 'https://git-codecommit.ap-south-1.amazonaws.com/v1/repos/Medyaan-PatientMonitoringSystemBackend'
            }
        }

        /**********************************************
         * 🔐 FIXED — GITLEAKS SECRET SCAN (NO ERRORS)
         **********************************************/
        stage('GitLeaks Secret Scan') {
            steps {
                script {
                    echo "🔍 Running GitLeaks to scan repository for secrets..."

                    sh """
                        export PATH=\$PATH:/usr/local/bin
                        gitleaks detect \
                            --source . \
                            --report-format json \
                            --report-path gitleaks-report.json \
                            --verbose || true
                    """

                    echo "📄 GitLeaks Report Output:"
                    sh "cat gitleaks-report.json || true"

                    archiveArtifacts artifacts: 'gitleaks-report.json', fingerprint: true

                    // FIX → Check if file exists
                    if (!fileExists('gitleaks-report.json')) {
                        error "❌ GitLeaks report not found — scan might have failed!"
                    }

                    // FIX → Correct jq for top-level array-based JSON
                    def secretCount = sh(
                        script: "jq 'length' gitleaks-report.json",
                        returnStdout: true
                    ).trim() as Integer

                    echo "🔢 GitLeaks Findings Count = ${secretCount}"

                    if (secretCount > 0) {
                        error "🛑 GitLeaks FAILED: ${secretCount} secrets detected!"
                    } else {
                        echo "✅ GitLeaks Passed: No hardcoded secrets found."
                    }
                }
            }
        }

        stage('Gradle Build') {
            steps {
                sh '''
                    chmod +x ./gradlew
                    ./gradlew clean build -x test
                '''
            }
        }

        stage('Unit Tests') {
            when { expression { params.RUN_TEST } }
            steps { sh './gradlew test' }
        }

        stage('SonarQube Analysis') {
            when { expression { params.RUN_SONAR } }
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                            -Dsonar.projectKey=Medyaan-Backend \
                            -Dsonar.sources=src/main/java \
                            -Dsonar.java.binaries=build/classes/java/main \
                            -Dsonar.sourceEncoding=UTF-8
                    """
                }
            }
        }

        stage('Quality Gate') {
            when { expression { params.RUN_QUALITY_GATE } }
            steps {
                script {
                    def qg = waitForQualityGate abortPipeline: false
                    if (qg.status != 'OK') {
                        error("❌ Quality Gate Failed!")
                    }
                }
            }
        }

        stage('OWASP Dependency Check') {
            when { expression { params.RUN_OWASP } }
            steps {
                dependencyCheck additionalArguments: "--scan ./ --disableYarnAudit --disableNodeAudit --data ${DC_DATA_DIR}",
                                 odcInstallation: 'DP'

                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'

                script {
                    if (!fileExists('dependency-check-report.xml')) {
                        error("⚠️ OWASP report not found.")
                    }

                    def report = readFile('dependency-check-report.xml')
                    if (report =~ /Critical|High|CRITICAL|HIGH/) {
                        error("❌ OWASP Scan Failed: HIGH/CRITICAL vulnerabilities found!")
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t ${ECR_REPO}:${IMAGE_TAG} .'
            }
        }

        /**********************************************
         * TRIVY IMAGE SCAN
         **********************************************/
        stage('Trivy Image Scan') {
            when { expression { params.RUN_TRIVY_IMAGE } }
            steps {
                script {

                    echo "🔍 Running Trivy Image Scan for Vulnerabilities + Secrets..."

                    sh """
                        trivy image --scanners vuln,secret --vuln-type os,library \
                        -f json -o trivy-image.json \
                        ${ECR_REPO}:${IMAGE_TAG} || true
                    """

                    archiveArtifacts artifacts: 'trivy-image.json', fingerprint: true

                    configFileProvider([configFile(fileId: 'TRIVY_REPORT_PY', targetLocation: 'trivy_report.py')]) {
                        sh 'python3 trivy_report.py trivy-image.json trivy-image-report.html || true'
                    }

                    archiveArtifacts artifacts: 'trivy-image-report.html'

                    def secrets = sh(script: "jq '[.Results[]?.Secrets? // [] | .[]] | length' trivy-image.json", returnStdout: true).trim() as Integer

                    if (secrets > 0) {
                        error("🛑 Trivy Image Secret Scan FAILED: Secrets Found = ${secrets}")
                    }

                    def critical = sh(script: "jq '[.Results[]?.Vulnerabilities? // [] | .[] | select(.Severity==\"CRITICAL\")] | length' trivy-image.json", returnStdout: true).trim() as Integer
                    def high     = sh(script: "jq '[.Results[]?.Vulnerabilities? // [] | .[] | select(.Severity==\"HIGH\")] | length' trivy-image.json", returnStdout: true).trim() as Integer
                    def medium   = sh(script: "jq '[.Results[]?.Vulnerabilities? // [] | .[] | select(.Severity==\"MEDIUM\")] | length' trivy-image.json", returnStdout: true).trim() as Integer
                    def low      = sh(script: "jq '[.Results[]?.Vulnerabilities? // [] | .[] | select(.Severity==\"LOW\")] | length' trivy-image.json", returnStdout: true).trim() as Integer

                    if (critical + high + medium + low > 0) {
                        error("❌ Trivy Image Scan Failed: Vulnerabilities found!")
                    }
                }
            }
        }

        /**********************************************
         * TRIVY FS SCAN
         **********************************************/
        stage('Trivy FS Scan') {
            when { expression { params.RUN_TRIVY_FS } }
            steps {
                script {

                    echo "🔍 Running Trivy FS Scan..."

                    sh """
                        trivy fs --scanners vuln,secret --vuln-type os,library \
                        -f json -o trivy-fs.json . || true
                    """

                    archiveArtifacts artifacts: 'trivy-fs.json', fingerprint: true

                    configFileProvider([configFile(fileId: 'TRIVY_REPORT_PY', targetLocation: 'trivy_report.py')]) {
                        sh 'python3 trivy_report.py trivy-fs.json trivy-fs-report.html || true'
                    }

                    archiveArtifacts artifacts: 'trivy-fs-report.html'

                    def secrets = sh(script: "jq '[.Results[]?.Secrets? // [] | .[]] | length' trivy-fs.json", returnStdout: true).trim() as Integer

                    if (secrets > 0) {
                        error("🛑 Trivy FS Secret Scan FAILED: Secrets Found = ${secrets}")
                    }

                    def critical = sh(script: "jq '[.Results[]?.Vulnerabilities? // [] | .[] | select(.Severity==\"CRITICAL\")] | length' trivy-fs.json", returnStdout: true).trim() as Integer
                    def high     = sh(script: "jq '[.Results[]?.Vulnerabilities? // [] | .[] | select(.Severity==\"HIGH\")] | length' trivy-fs.json", returnStdout: true).trim() as Integer
                    def medium   = sh(script: "jq '[.Results[]?.Vulnerabilities? // [] | .[] | select(.Severity==\"MEDIUM\")] | length' trivy-fs.json", returnStdout: true).trim() as Integer
                    def low      = sh(script: "jq '[.Results[]?.Vulnerabilities? // [] | .[] | select(.Severity==\"LOW\")] | length' trivy-fs.json", returnStdout: true).trim() as Integer

                    if (critical + high + medium + low > 0) {
                        error("❌ Trivy FS Scan Failed: Vulnerabilities Found!")
                    }
                }
            }
        }

        stage('Docker Push to ECR') {
            steps {
                withAWS(region: AWS_REGION, credentials: 'medyaan-ecr-credentials') {
                    sh '''
                        aws ecr get-login-password --region ${AWS_REGION} \
                          | docker login --username AWS --password-stdin ${ECR_REPO}

                        docker push ${ECR_REPO}:${IMAGE_TAG}
                        docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_REPO}:latest
                        docker push ${ECR_REPO}:latest
                    '''
                }
            }
        }

        stage('Deploy using Docker Compose') {
            when { expression { params.RUN_DEPLOY } }
            steps {
                sh '''
                    cd ${DEPLOY_DIR}
                    docker-compose down || true
                    docker-compose pull
                    docker-compose up -d
                '''
            }
        }
    }

    post {
        success { echo "✅ Pipeline completed successfully!" }
        failure { echo "❌ PIPELINE FAILED – Check GitLeaks / Trivy / OWASP / Sonar results!" }
        always  { echo "📄 Reports generated and archived." }
    }
}

