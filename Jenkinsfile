pipeline {
    agent {
        docker {
            image 'gdiaz90/node-with-docker-cli:22'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
            reuseNode true
        }
    }

    environment {
        IMAGE_NAME = "gdiaz90/backend-test"
    }

    stages {
        stage('Instalación de dependencias') {
            steps { sh 'npm install' }
        }

        stage('Pruebas automatizadas') {
            steps { sh 'npm run test:cov' }
        }

        stage('Construcción de aplicación') {
            steps { sh 'npm run build' }
        }

        stage('Quality Assurance') {
            agent {
                docker {
                    image 'sonarsource/sonar-scanner-cli'
                    args "-v ${env.WORKSPACE}:${env.WORKSPACE} --network=dockercompose_devnet"
                }
            }
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh "sonar-scanner -Dsonar.projectKey=backend-test -Dsonar.sources=src -Dsonar.tests=src -Dsonar.javascript.lcov.reportPaths=${env.WORKSPACE}/coverage/lcov.info"
                }
            }
        }

        stage('Quality Gate'){
            steps {
                timeout(time: 30, unit: 'SECONDS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Empaquetado y push Docker') {
            steps {
                script {
                    def app = docker.build("${IMAGE_NAME}:${BUILD_NUMBER}")

                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-credentials') {
                        app.push()          
                        app.push("gdd")     
                    }

                    docker.withRegistry('http://localhost:8082', 'nexus-credentials') {
                        sh "docker tag ${IMAGE_NAME}:${BUILD_NUMBER} localhost:8082/${IMAGE_NAME}:${BUILD_NUMBER}"
                        sh "docker tag ${IMAGE_NAME}:${BUILD_NUMBER} localhost:8082/${IMAGE_NAME}:latest"

                        sh "docker push localhost:8082/${IMAGE_NAME}:${BUILD_NUMBER}"
                        sh "docker push localhost:8082/${IMAGE_NAME}:latest"
                    }
                }
            }
        }
    }
}
