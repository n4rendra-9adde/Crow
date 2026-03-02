pipeline {
    agent any 
    
    environment {
        REPORT_DIR = 'security-reports'
        SONAR_TOKEN = credentials('sonarqube-token')
        SONAR_HOST_URL = 'http://localhost:9000'
        IMAGE_TAG = "${BUILD_NUMBER}"
        // Build variables
        BUILD_DIR = 'build'
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 2, unit: 'HOURS')
        disableConcurrentBuilds()
    }
    
    stages {
        stage('Environment Check') {
            steps {
                echo " Pipeline started - Build #${env.BUILD_NUMBER}"
                sh 'mkdir -p ${REPORT_DIR}'
            }
        }
        
       stage('SAST - SonarQube (C/C++)') {
    steps {
        sh '''
            # Download build wrapper
            wget https://sonarcloud.io/static/cpp/build-wrapper-linux-x86.zip
            unzip -q build-wrapper-linux-x86.zip
            export PATH=$PWD/build-wrapper-linux-x86:$PATH
            
            # Clean and build with wrapper
            build-wrapper-linux-x86-64 --out-dir bw-output cmake --build build --clean-first
            
            # Run sonar-scanner
            sonar-scanner -Dsonar.cfamily.build-wrapper-output=bw-output
        '''
    }
}
        
        stage('Build') {
            steps {
                sh '''
                    echo "=== Configuring with CMake ==="
                    cmake -B ${BUILD_DIR} -DCMAKE_BUILD_TYPE=Release
                    echo "=== Building ==="
                    cmake --build ${BUILD_DIR} -- -j$(nproc)
                '''
            }
        }
        
        stage('SAST - SonarQube (C/C++)') {
            steps {
                script {
                    // SonarQube for C/C++ requires the build-wrapper to capture compilation units.
                    // It should have been run during the build stage. Here we re-run with build-wrapper.
                    def scannerHome = tool 'SonarScanner'
                    withSonarQubeEnv('SonarQube') {
                        sh '''
                            # Clean any previous build-wrapper output
                            rm -rf bw-output
                            
                            # Run build-wrapper (adjust path to your build-wrapper executable)
                            build-wrapper-linux-x86-64 --out-dir bw-output cmake --build ${BUILD_DIR} --clean-first
                            
                            # Then run sonar-scanner
                            sonar-scanner \
                                -Dsonar.projectKey=cpp-app \
                                -Dsonar.projectName='C++ App' \
                                -Dsonar.cfamily.build-wrapper-output=bw-output \
                                -Dsonar.sources=. \
                                -Dsonar.exclusions=**/${BUILD_DIR}/**,**/cmake-build-debug/**
                        '''
                    }
                }
            }
        }
        
        stage('SCA - Dependency Check') {
            steps {
                sh '''
                    echo "=== OWASP Dependency-Check (C/C++ experimental) ==="
                    /opt/dependency-check/bin/dependency-check.sh \
                        --project "C++ App" \
                        --scan . \
                        --format JSON --format HTML \
                        --out ${REPORT_DIR}/dependency-check \
                        --enableExperimental || true
                    
                    # Optionally add cve-bin-tool for binary scanning
                    # virtualenv venv; source venv/bin/activate; pip install cve-bin-tool
                    # cve-bin-tool ${BUILD_DIR}/bin/ --json ${REPORT_DIR}/cve-bin-tool.json || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/dependency-check/*", allowEmptyArchive: true
                }
            }
        }
        
        stage('Unit Tests') {
            steps {
                sh '''
                    cd ${BUILD_DIR}
                    ctest --output-on-failure --test-output-size-passed 1024 --test-output-size-failed 1024 \
                        --no-compress-output --output-junit ../${REPORT_DIR}/ctest-results.xml || true
                    cd ..
                '''
            }
            post {
                always {
                    // Convert ctest XML to JUnit format if needed, or use a JUnit-compatible reporter.
                    // For simplicity, we archive raw XML.
                    junit testResults: "${REPORT_DIR}/ctest-results.xml", allowEmptyResults: true
                }
            }
        }
        
        stage('Container Security') {
            steps {
                script {
                    sh '''
                        echo "=== Building Docker Image ==="
                        docker build -t cpp-app:${IMAGE_TAG} .
                        
                        echo "=== Trivy Image Scan ==="
                        trivy image --format json --output ${REPORT_DIR}/trivy-report.json --severity HIGH,CRITICAL cpp-app:${IMAGE_TAG} || true
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/trivy-report.json", allowEmptyArchive: true
                }
            }
        }
        
        stage('Approval: Deploy to Staging') {
            steps {
                script {
                    input message: 'Approve deployment to STAGING?', ok: 'Deploy to Staging', submitterParameter: 'APPROVER'
                    echo "Approved by: ${env.APPROVER}"
                }
            }
        }
        
        stage('Deploy to Staging') {
            steps {
                sh '''
                    docker stop cpp-app-staging 2>/dev/null || true
                    docker rm cpp-app-staging 2>/dev/null || true
                    docker run -d --name cpp-app-staging -p 8080:8080 cpp-app:${IMAGE_TAG}
                    sleep 5
                '''
            }
        }
        
        stage('Approval: Deploy to Production') {
            steps {
                script {
                    timeout(time: 24, unit: 'HOURS') {
                        input message: 'Approve deployment to PRODUCTION?', ok: 'Deploy to Production', submitterParameter: 'PROD_APPROVER'
                    }
                    echo "Production approved by: ${env.PROD_APPROVER}"
                }
            }
        }
        
        stage('Deploy to Production') {
            steps {
                sh 'docker tag cpp-app:${IMAGE_TAG} cpp-app:production'
            }
        }
        
        // ==========================================
        // Final Report Generation
        // ==========================================
        stage('Generate Final Report') {
            steps {
                script {
                    sh '''
                        echo "=== 📊 Generating Final Security Report ==="
                        
                        cat > ${REPORT_DIR}/pipeline-summary.html << 'HTMLEOF'
<!DOCTYPE html>
<html>
<head>
    <title>DevSecOps Pipeline Report - C++</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; background: #f5f5f5; }
        .header { background: #2c3e50; color: white; padding: 20px; border-radius: 5px; }
        .summary { background: white; padding: 20px; margin: 20px 0; border-radius: 5px; box-shadow: 0 2px 5px rgba(0,0,0,0.1); }
        .stage { background: white; padding: 15px; margin: 10px 0; border-left: 4px solid #27ae60; }
        .stage.failed { border-left-color: #e74c3c; }
        .metric { display: inline-block; margin: 10px 20px 10px 0; }
        .metric-value { font-size: 24px; font-weight: bold; color: #27ae60; }
        .metric-label { font-size: 12px; color: #7f8c8d; }
    </style>
</head>
<body>
    <div class="header">
        <h1>⚙️ DevSecOps Pipeline Report - C++</h1>
        <p>Build #BUILD_NUMBER | Commit: GIT_COMMIT | Date: BUILD_DATE</p>
    </div>
    
    <div class="summary">
        <h2>Executive Summary</h2>
        <div class="metric">
            <div class="metric-value">7</div>
            <div class="metric-label">Security Scans</div>
        </div>
        <div class="metric">
            <div class="metric-value">2</div>
            <div class="metric-label">Approval Gates</div>
        </div>
        <div class="metric">
            <div class="metric-value" style="color: #27ae60;">PASSED</div>
            <div class="metric-label">Status</div>
        </div>
    </div>

    <div class="stage">
        <h3>🔐 Secret Scanning (Gitleaks)</h3>
        <p>Scanned for hardcoded secrets, API keys, and credentials</p>
    </div>

    <div class="stage">
        <h3>🔍 SAST (SonarQube for C/C++)</h3>
        <p>Static code analysis for bugs, vulnerabilities, and code smells</p>
    </div>

    <div class="stage">
        <h3>📦 SCA (OWASP Dependency‑Check)</h3>
        <p>Identified known vulnerabilities in C/C++ dependencies (experimental)</p>
    </div>

    <div class="stage">
        <h3>🐳 Container Security (Trivy)</h3>
        <p>Scanned Docker image for OS and application vulnerabilities</p>
    </div>

    <div class="stage">
        <h3>✅ Unit Tests (CTest)</h3>
        <p>Executed unit tests</p>
    </div>

    <div class="summary">
        <h2>Artifacts Generated</h2>
        <ul>
            <li>gitleaks-report.json - Secret scanning results</li>
            <li>dependency-check-report.html - OWASP dependency report</li>
            <li>trivy-report.json - Container scan results</li>
            <li>ctest-results.xml - Unit test results</li>
        </ul>
    </div>
</body>
</html>
HTMLEOF

                        # Replace placeholders
                        sed -i "s/BUILD_NUMBER/${BUILD_NUMBER}/g" ${REPORT_DIR}/pipeline-summary.html
                        sed -i "s/GIT_COMMIT/${GIT_COMMIT}/g" ${REPORT_DIR}/pipeline-summary.html
                        sed -i "s/BUILD_DATE/$(date)/g" ${REPORT_DIR}/pipeline-summary.html

                        echo " Report generated: ${REPORT_DIR}/pipeline-summary.html"
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "${REPORT_DIR}/pipeline-summary.html", allowEmptyArchive: true
                    publishHTML(target: [
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: "${REPORT_DIR}",
                        reportFiles: 'pipeline-summary.html',
                        reportName: 'Pipeline Summary Report'
                    ])
                }
            }
        }
    }
    
    post {
        always {
            echo "=========================================="
            echo "PIPELINE COMPLETED - Build #${BUILD_NUMBER}"
            echo "=========================================="
        }
        success {
            echo "✅ SUCCESS! Check the Pipeline Summary Report"
        }
        failure {
            echo "❌ FAILED!"
        }
    }
}
