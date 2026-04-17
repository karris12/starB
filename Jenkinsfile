pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME   = tool 'sonar-scanner'
        APP_NAME       = 'starbucks'
        DOCKER_IMAGE   = "agodzo/${APP_NAME}:latest"
        GITHUB_REPO    = 'https://github.com/karris12/starbucks.git'
        CONTAINER_NAME = 'starbucks-app'
        HOST_PORT      = '30000'
        APP_PORT       = '3000'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Git Checkout') {
            steps {
                git branch: 'main', url: "${GITHUB_REPO}"
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=${APP_NAME} \
                        -Dsonar.projectKey=${APP_NAME}
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('Install NPM Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit',
                                odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy File System Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${APP_NAME} ."
            }
        }

        stage('Tag & Push to DockerHub') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker tag ${APP_NAME} ${DOCKER_IMAGE}"
                        sh "docker push ${DOCKER_IMAGE}"
                    }
                }
            }
        }

        stage('Docker Scout Image Scan') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        // Quick overview of the image
                        sh "docker-scout quickview ${DOCKER_IMAGE}"

                        // CVE vulnerability scan
                        sh "docker-scout cves ${DOCKER_IMAGE}"

                        // Remediation recommendations
                        sh "docker-scout recommendations ${DOCKER_IMAGE}"
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image ${DOCKER_IMAGE} > trivyimage.txt"
            }
        }

        stage('Deploy to Container') {
            steps {
                script {
                    // Remove existing container if already running
                    sh """
                        docker stop ${CONTAINER_NAME} || true
                        docker rm   ${CONTAINER_NAME} || true
                        docker run -d \
                            --name ${CONTAINER_NAME} \
                            -p ${HOST_PORT}:${APP_PORT} \
                            ${DOCKER_IMAGE}
                    """
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivyfs.txt, trivyimage.txt', allowEmptyArchive: true

            emailext (
                attachLog: true,
                to: 'rooseveltaws@gmail.com',
                subject: "Pipeline [${APP_NAME}] - Build #${BUILD_NUMBER} - ${currentBuild.currentResult}",
                mimeType: 'text/html',
                attachmentsPattern: 'trivyfs.txt, trivyimage.txt',
                body: """
                    <html>
                    <body>
                        <h2 style="color: #2c3e50;">Jenkins Pipeline Report</h2>

                        <div style="background-color: #FFA07A; padding: 10px; margin-bottom: 10px; border-radius: 5px;">
                            <p style="color: white; font-weight: bold;">
                                Project: ${env.JOB_NAME}
                            </p>
                        </div>

                        <div style="background-color: #90EE90; padding: 10px; margin-bottom: 10px; border-radius: 5px;">
                            <p style="color: white; font-weight: bold;">
                                Build Number: ${env.BUILD_NUMBER}
                            </p>
                        </div>

                        <div style="background-color: #87CEEB; padding: 10px; margin-bottom: 10px; border-radius: 5px;">
                            <p style="color: white; font-weight: bold;">
                                Build Status: ${currentBuild.currentResult}
                            </p>
                        </div>

                        <div style="background-color: #DDA0DD; padding: 10px; margin-bottom: 10px; border-radius: 5px;">
                            <p style="color: white; font-weight: bold;">
                                Docker Image: ${DOCKER_IMAGE}
                            </p>
                        </div>

                        <div style="background-color: #F0E68C; padding: 10px; margin-bottom: 10px; border-radius: 5px;">
                            <p style="color: #333; font-weight: bold;">
                                Duration: ${currentBuild.durationString}
                            </p>
                        </div>

                        <div style="background-color: #778899; padding: 10px; margin-bottom: 10px; border-radius: 5px;">
                            <p style="color: white; font-weight: bold;">
                                Build URL: <a href="${env.BUILD_URL}" style="color: #FFD700;">${env.BUILD_URL}</a>
                            </p>
                        </div>

                        <p style="color: #555;">Trivy scan reports attached. Check console output for Docker Scout analysis.</p>
                    </body>
                    </html>
                """
            )
        }

        success {
            echo "✅ ${APP_NAME} deployed successfully on port ${HOST_PORT}!"
        }

        failure {
            echo "❌ Pipeline failed — review Docker Scout and Trivy reports."
        }
    }
}
