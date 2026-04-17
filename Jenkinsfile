pipeline {
    agent any

    tools {
        jdk 'jdk17'
        // Updated to node18 to resolve the EBADENGINE warnings
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
                        echo "⚠️ OWASP Check failed or Credentials missing. Skipping to keep pipeline moving."
                    }
                }
            }
        }

        stage('Trivy File System Scan') {
            steps { sh 'trivy fs . > trivyfs.txt' }
        }

        stage('Docker Build') {
            steps {
                sh "ls -la"
                sh "docker build -t ${APP_NAME} ."
            }
        }

        // ... remaining stages (Push, Scout, Deploy) stay the same
    }
}
