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

        stage('Deploy to Container') {
            steps {
                sh """
                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true
                    docker run -d --name ${CONTAINER_NAME} -p ${HOST_PORT}:${APP_PORT} ${DOCKER_IMAGE}
                """
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivyfs.txt, trivyimage.txt', allowEmptyArchive: true
            emailext (
                attachLog: true,
                to: 'rooseveltaws@gmail.com',
                subject: "Status: ${currentBuild.currentResult} - Build #${BUILD_NUMBER}",
                body: "Pipeline finished for ${APP_NAME}. Check results here: ${env.BUILD_URL}"
            )
        }
    }
}
