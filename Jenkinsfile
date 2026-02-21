pipeline {
    agent any

    tools {
        jdk 'jdk21'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_NAME = "pragadeesh2102/amazon"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage("Clean Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout Code") {
            steps {
                git branch: 'main', url: 'https://github.com/pragadeesh2102/amazon-Devsecops.git'
            }
        }

        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                    ${SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectName=amazon \
                    -Dsonar.projectKey=amazon
                    """
                }
            }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("Install Dependencies") {
            steps {
                sh "npm install"
            }
        }

        stage("OWASP FS Scan") {
    steps {
        script {
            withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    dependencyCheck additionalArguments: """
                        --scan ./ 
                        --disableYarnAudit 
                        --disableNodeAudit 
                        --nvdApiKey=${NVD_API_KEY}
                    """,
                    odcInstallation: 'dp-check'
                }
            }
        }
    }
}

        stage("Publish OWASP Report") {
            steps {
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage("Trivy File Scan") {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }

        stage("Build Docker Image") {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage("Trivy Image Scan") {
            steps {
                sh """
                trivy image --severity HIGH,CRITICAL \
                --exit-code 1 ${IMAGE_NAME}:${IMAGE_TAG} || true
                """
            }
        }

        stage("Push Docker Image") {
            steps {
                withCredentials([string(credentialsId: 'docker-cred', variable: 'DOCKER_PWD')]) {
                    sh """
                    docker login -u pragadeesh2102 -p ${DOCKER_PWD}
                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                    docker push ${IMAGE_NAME}:latest
                    """
                }
            }
        }

        stage("Deploy Container") {
            steps {
                sh """
                docker rm -f amazon || true
                docker run -d -p 80:80 --name amazon ${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline executed successfully!"
        }

        failure {
            echo "❌ Pipeline failed. Check logs."
        }

        always {
            emailext (
                subject: "Build ${currentBuild.currentResult}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                <h3>DevSecOps Pipeline Status</h3>
                <p><b>Project:</b> ${env.JOB_NAME}</p>
                <p><b>Build:</b> #${env.BUILD_NUMBER}</p>
                <p><b>Status:</b> ${currentBuild.currentResult}</p>
                <p><b>URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                """,
                to: 'pragadeeshuthamarajan@gmail.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivyfs.txt,dependency-check-report.xml'
            )
        }
    }
}
