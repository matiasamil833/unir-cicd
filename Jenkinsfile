pipeline {
    agent {
        label "docker"
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_REPOSITORY= 'matiasamil833'
        IMAGE_TAG = "v${env.BUILD_NUMBER}"
        IMAGE_NAME = "${env.JOB_NAME}"
        EMAIL_NOTIFICATION = "matias.amil833@comunidadunir.net"
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/matiasamil833/unir-cicd.git'
            }
        }
        stage('Build') {
            steps {
                echo 'Building stage!'
                sh 'make build'
            }
        }
        stage('Run Unit Test') {
            steps {
                sh 'make test-unit'
                archiveArtifacts artifacts: 'results/*.xml'
            }
        }
        stage('Run API Test') {
            steps {
                sh 'make test-api'
                archiveArtifacts artifacts: 'results/api_result.xml'
            }
        }
        stage('Run E2E Test') {
            steps {
                sh 'make test-e2e'
                archiveArtifacts artifacts: 'results/cypress_result.xml'
            }
        }
        stage ('Static Analysis')
        {
            parallel {
                stage('Style') {
                    steps {
                        echo 'Validate style'
                        sh "make pylint"
                        archiveArtifacts artifacts: 'results/pylint_result.txt'
                    }
                }
                stage('Scan Vulnerabilities') {
                    steps {
                        echo "Scan Vulnerabilities"
                        sh 'docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v "$(pwd)":/app -v /root/.cache/trivy:/root/.cache aquasec/trivy fs --format json --output trivy-report.json --severity HIGH,CRITICAL --exit-code 1 /app'
                        archiveArtifacts artifacts: 'trivy-report.json', allowEmptyArchive: true
                    }
                }
                stage('SonarQube Analysis') {
                    steps {
                        withSonarQubeEnv('sonarqube') {
                            sh """
                                ${SCANNER_HOME}/bin/sonar-scanner \
                                -Dsonar.projectKey=unir-cicd \
                                -Dsonar.projectName="Unir CICD" \
                                -Dsonar.sources=. \
                                -Dsonar.exclusions=**/*.js,**/*.jsx,**/*.ts,**/*.tsx,**/*.vue,**/*.mjs,**/*.cjs,**/*.html
                            """
                        }
                    }
                }
            }
        }
        stage('Build Docker Image'){
            steps {
                echo "Docker build"
                sh "docker build -t ${DOCKER_REPOSITORY}/${IMAGE_NAME}:${IMAGE_TAG} -f ./Dockerfile ."
                sh "docker tag ${DOCKER_REPOSITORY}/${IMAGE_NAME}:${IMAGE_TAG} ${DOCKER_REPOSITORY}/${IMAGE_NAME}:latest"
            }
        }
        stage('Publish Artifact') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
                     script {
                        sh "echo '${DOCKER_PASSWORD}' | docker login -u '${DOCKER_USER}' --password-stdin"
                        sh "docker push ${DOCKER_REPOSITORY}/${IMAGE_NAME}:${IMAGE_TAG}"
                        sh "docker push ${DOCKER_REPOSITORY}/${IMAGE_NAME}:latest"
                        sh "docker logout"
                    }
                }
            }
        }
    }
    post {
        always {
            echo "Esto se ejecuta SIEMPRE (éxito, fallo, etc)"
            junit 'results/*_result.xml'
            publishHTML(target: [
                reportDir: 'results/',
                reportFiles: 'index.html',
                reportName: 'Reporte HTML'
            ])
        }

        success {
            echo "Pipeline finalizó con ÉXITO"
        }

        failure {
            echo "Pipeline ${env.JOB_NAME} build ${env.BUILD_NUMBER} FALLÓ"
            mail to: "${env.EMAIL_NOTIFICATION}",
                 subject: "❌ FALLÓ el Pipeline: ${currentBuild.fullDisplayName}",
                 body: """¡Atención! El despliegue o la construcción ha fallado.
                          
                          - Proyecto: ${env.JOB_NAME}
                          - Build: #${env.BUILD_NUMBER}
                          - Por favor, revisa los logs en: ${env.BUILD_URL}console"""
        }

        cleanup {
            // Borramos las imagenes docker de la ejecución.
            sh "docker rmi ${DOCKER_REPOSITORY}/${IMAGE_NAME}:${IMAGE_TAG} || true"
            sh "docker rmi ${DOCKER_REPOSITORY}/${IMAGE_NAME}:latest || true"
            cleanWs()
        }
    }
}
