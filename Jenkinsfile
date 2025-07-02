pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/bodkekarbalaji95/DevSecOps-Project.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Netflix \
                        -Dsonar.projectKey=Netflix'''
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

        stage('Install Dependencies') {
            steps {
                sh '''
                    npm install
                    npm i --package-lock-only
                    npm audit fix || true
                    npm audit fix --force
                    npm fund
                '''
            }
        }

        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'OWASP Dependency-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'

                
            }
            post {
                always {
                    archiveArtifacts artifacts: '**/dependency-check-report.xml', allowEmptyArchive: true
                }
            }
        }

        stage('TRIVY FS SCAN') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivyfs.txt', allowEmptyArchive: true
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'tmdb-api', variable: 'API_KEY')]) {
                        withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                            sh "docker build --build-arg TMDB_V3_API_KEY=$API_KEY -t netflix ."
                            sh "docker tag netflix bodkekarbalaji95/netflix:latest"
                            sh "docker push bodkekarbalaji95/netflix:latest"
                        }
                    }
                }
            }
        }

        stage('TRIVY Image SCAN') {
            steps {
                sh 'trivy image bodkekarbalaji95/netflix:latest > trivyimage.txt'
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivyimage.txt', allowEmptyArchive: true
                }
            }
        }

        stage('Deploy to Container') {
            steps {
                sh 'docker run -d --name netflix -p 8081:80 bodkekarbalaji95/netflix:latest'
            }
        }
    }
}
