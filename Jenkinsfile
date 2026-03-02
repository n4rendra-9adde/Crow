pipeline {
    agent any
    
    parameters {
        choice(
            name: 'APP_TYPE',
            choices: ['Microservice (Docker)', 'Monolithic (Binary)'],
            description: 'Application type - determines deployment method'
        )
        string(
            name: 'BINARY_NAME',
            defaultValue: 'myapp',
            description: 'Name of your compiled binary (for monolithic apps)'
        )
    }
    
    environment {
        REPORT_DIR = 'security-reports'
        BUILD_DIR = 'build'
        DEPLOY_DIR = '/opt/myapp'  // Where to deploy monolithic binary
    }
    
    stages {
        stage('Build') {
            steps {
                sh '''
                    cmake -B ${BUILD_DIR} -DCMAKE_BUILD_TYPE=Release
                    cmake --build ${BUILD_DIR} -- -j$(nproc)
                '''
            }
        }
        
        stage('Deploy - Microservice') {
            when {
                expression { params.APP_TYPE == 'Microservice (Docker)' }
            }
            steps {
                script {
                    // Only for Docker-based apps
                    sh '''
                        docker build -t cpp-app:${BUILD_NUMBER} .
                        docker run -d --name cpp-app -p 8080:8080 cpp-app:${BUILD_NUMBER}
                    '''
                }
            }
        }
        
        stage('Deploy - Monolithic') {
            when {
                expression { params.APP_TYPE == 'Monolithic (Binary)' }
            }
            steps {
                script {
                    // Interactive deployment for monolithic
                    def deployTarget = input(
                        id: 'deploy-target',
                        message: 'Select deployment target for monolithic app',
                        parameters: [
                            choice(
                                name: 'TARGET',
                                choices: ['Local Directory', 'Remote Server (SSH)', 'Systemd Service', 'Skip Deployment'],
                                description: 'Where to deploy the binary?'
                            )
                        ]
                    )
                    
                    if (deployTarget == 'Local Directory') {
                        sh """
                            echo "Deploying binary locally..."
                            sudo mkdir -p ${DEPLOY_DIR}
                            sudo cp ${BUILD_DIR}/${params.BINARY_NAME} ${DEPLOY_DIR}/
                            sudo chmod +x ${DEPLOY_DIR}/${params.BINARY_NAME}
                            
                            # Create systemd service file
                            sudo tee /etc/systemd/system/${params.BINARY_NAME}.service > /dev/null << EOF
[Unit]
Description=${params.BINARY_NAME} Service
After=network.target

[Service]
Type=simple
User=jenkins
WorkingDirectory=${DEPLOY_DIR}
ExecStart=${DEPLOY_DIR}/${params.BINARY_NAME}
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
                            
                            echo "✅ Binary deployed to ${DEPLOY_DIR}"
                            echo "To start: sudo systemctl start ${params.BINARY_NAME}"
                        """
                    } else if (deployTarget == 'Remote Server (SSH)') {
                        // Use SSH to deploy
                        sshPublisher(
                            publishers: [
                                sshPublisherDesc(
                                    configName: 'production-server',
                                    transfers: [
                                        sshTransfer(
                                            sourceFiles: "${BUILD_DIR}/${params.BINARY_NAME}",
                                            remoteDirectory: "/opt/apps",
                                            execCommand: "chmod +x /opt/apps/${params.BINARY_NAME} && systemctl restart ${params.BINARY_NAME}"
                                        )
                                    ]
                                )
                            ]
                        )
                    } else if (deployTarget == 'Systemd Service') {
                        sh """
                            # Install and start as systemd service
                            sudo cp ${BUILD_DIR}/${params.BINARY_NAME} /usr/local/bin/
                            sudo systemctl daemon-reload
                            sudo systemctl enable ${params.BINARY_NAME}
                            sudo systemctl restart ${params.BINARY_NAME}
                        """
                    }
                }
            }
        }
    }
}
