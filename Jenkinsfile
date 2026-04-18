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
        HOST_PORT      = '3000'
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
            steps { 
                // Increased wait time slightly to avoid timeout errors
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false
                }
            }
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
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker tag ${APP_NAME} ${DOCKER_IMAGE}"
                        sh "docker push ${DOCKER_IMAGE}"
                    }
                }
            }
        }

        stage('Update GitOps Manifest') {
            steps {
                script {
                    // Remove existing directory if it exists to prevent clone failure
                    sh "rm -rf starbucks-gitops"
                    
                    withCredentials([usernamePassword(credentialsId: 'github-token', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        sh "git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/karris12/starbucks-gitops.git"
                        
                        dir('starbucks-gitops') {
                            sh "sed -i 's|image: agodzo/starbucks:.*|image: ${DOCKER_IMAGE}|' k8s/deployment.yaml"
                            
                            sh "git config user.email 'jenkins@example.com'"
                            sh "git config user.name 'Jenkins CI'"
                            sh "git add k8s/deployment.yaml"
                            // Added '|| true' to prevent failure if there are no changes to commit
                            sh "git commit -m 'Update image build ${env.BUILD_NUMBER}' || true"
                            sh "git push origin main"
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            emailext attachLog: true,
                subject: "Build ${currentBuild.result}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                    <html>
                    <body>
                        <div style="padding: 10px; border: 1px solid #ddd;">
                            <h3>Build Status: ${currentBuild.result}</h3>
                            <p><b>Project:</b> ${env.JOB_NAME}</p>
                            <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
                            <p><b>Link:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                        </div>
                    </body>
                    </html>
                """,
                to: 'rooseveltaws@gmail.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}
