pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node18' 
    }

    environment {
        SCANNER_HOME   = tool 'sonar-scanner'
        APP_NAME       = 'starbucks'
        DOCKER_IMAGE   = "agodzo/${APP_NAME}:latest"
        GITHUB_REPO    = 'https://github.com/karris12/starB.git'
        CONTAINER_NAME = 'starbucks-app'
        HOST_PORT      = '30000'
        APP_PORT       = '3000'
    }

    stages {
        stage('Clean Workspace') {
            steps { cleanWs() }
        }

        stage('Git Checkout') {
            steps { git branch: 'main', url: "${GITHUB_REPO}" }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh "${SCANNER_HOME}/bin/sonar-scanner -Dsonar.projectName=${APP_NAME} -Dsonar.projectKey=${APP_NAME}"
                }
            }
        }

        stage('Quality Gate') {
            steps { waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token' }
        }

        stage('Install NPM Dependencies') {
            steps { sh 'npm install' }
        }

        stage('OWASP Dependency Check') {
            steps {
                script {
                    try {
                        withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                            dependencyCheck additionalArguments: "--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey ${NVD_API_KEY}",
                                            odcInstallation: 'DP-Check'
                            dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                        }
                    } catch (Exception e) {
                        echo "⚠️ Skipping OWASP Check: Scan failed or API key issue."
                    }
                }
            }
        }

        stage('Trivy File System Scan') {
            steps { sh 'trivy fs . > trivyfs.txt' }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${APP_NAME} ."
            }
        }

        stage('Tag & Push to DockerHub') {
            steps {
                script {
                    // Removed toolName to use system Docker client
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker tag ${APP_NAME} ${DOCKER_IMAGE}"
                        sh "docker push ${DOCKER_IMAGE}"
                    }
                }
            }
        }

        stage('Docker Scout Scan') {
            steps {
                script {
                    // Removed toolName to use system Docker client
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker-scout quickview ${DOCKER_IMAGE}"
                        sh "docker-scout cves ${DOCKER_IMAGE}"
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps { sh "trivy image ${DOCKER_IMAGE} > trivyimage.txt" }
        }

        stage('Update GitOps Manifest') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'github-token', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        // 1. Clone the GitOps repo
                        sh "git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/karris12/starbucks-gitops.git"
                        
                        dir('starbucks-gitops') {
                            // 2. Update the image tag in deployment.yaml (using sed)
                            sh "sed -i 's|image: agodzo/starbucks:.*|image: ${DOCKER_IMAGE}|' k8s/deployment.yaml"
                            
                            // 3. Commit and Push back to GitHub
                            sh "git config user.email 'jenkins@example.com'"
                            sh "git config user.name 'Jenkins CI'"
                            sh "git add k8s/deployment.yaml"
                            sh "git commit -m 'Update starbucks image to build ${env.BUILD_NUMBER}'"
                            sh "git push origin main"
                        }
                    }
                }
            }
        }

     post {
    always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: """
                <html>
                <body>
                    <div style="background-color: #FFA07A; padding: 10px; margin-bottom: 10px;">
                        <p style="color: white; font-weight: bold;">Project: ${env.JOB_NAME}</p>
                    </div>
                    <div style="background-color: #90EE90; padding: 10px; margin-bottom: 10px;">
                        <p style="color: white; font-weight: bold;">Build Number: ${env.BUILD_NUMBER}</p>
                    </div>
                    <div style="background-color: #87CEEB; padding: 10px; margin-bottom: 10px;">
                        <p style="color: white; font-weight: bold;">URL: ${env.BUILD_URL}</p>
                    </div>
                </body>
                </html>
            """,
            to: 'rooseveltaws@gmail.com',
            mimeType: 'text/html',
            attachmentsPattern: 'trivy.txt'
        }
    }
}

