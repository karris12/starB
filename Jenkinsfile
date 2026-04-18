pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node18' 
    }

    environment {
        SCANNER_HOME   = tool 'sonar-scanner'
        APP_NAME       = 'starbucks'
        // Using BUILD_NUMBER ensures a unique tag for every deployment
        DOCKER_IMAGE   = "agodzo/${APP_NAME}:${BUILD_NUMBER}"
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
                // Build with the unique tag
                sh "docker build -t ${DOCKER_IMAGE} ."
            }
        }

        stage('Tag & Push to DockerHub') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push ${DOCKER_IMAGE}"
                        // Also push a 'latest' tag for convenience
                        sh "docker tag ${DOCKER_IMAGE} agodzo/${APP_NAME}:latest"
                        sh "docker push agodzo/${APP_NAME}:latest"
                    }
                }
            }
        }

        stage('Update GitOps Manifest') {
            steps {
                script {
                    // Clean up any old clones before starting
                    sh "rm -rf starbucks-gitops"
                    
                    // We use the 'github-token' ID you created in Jenkins
                    withCredentials([string(credentialsId: 'github-token', variable: 'GH_TOKEN')]) {
                        // Clone using the token for authentication
                        sh "git clone https://${GH_TOKEN}@github.com/karris12/starbucks-gitops.git"
                        
                        dir('starbucks-gitops') {
                            // Update the image tag in your K8s deployment file
                            sh "sed -i 's|image: agodzo/starbucks:.*|image: ${DOCKER_IMAGE}|' k8s/deployment.yaml"
                            
                            sh "git config user.email 'jenkins@example.com'"
                            sh "git config user.name 'Jenkins CI'"
                            sh "git add k8s/deployment.yaml"
                            sh "git commit -m 'Update image to build ${env.BUILD_NUMBER}' || echo 'No changes to commit'"
                            sh "git push https://${GH_TOKEN}@github.com/karris12/starbucks-gitops.git main"
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
                            <p><b>Docker Image:</b> ${DOCKER_IMAGE}</p>
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
