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
                    sh "$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix -Dsonar.projectKey=Netflix -Dsonar.sources=."
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
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_KEY')]) {
                    sh '''
                        export PATH="/var/lib/jenkins/tools/dependency-check/dependency-check/bin:$PATH"
                        dependency-check.sh \
                            --scan . \
                            --format ALL \
                            --out ./reports \
                            --nvdApiKey $NVD_KEY \
                            --nvdApiDelay 6000 \
                            --disableYarnAudit \
                            --disableNodeAudit || true
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'reports/*.*', allowEmptyArchive: true
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
                withCredentials([
                    string(credentialsId: 'Docker', variable: 'DOCKER_TOKEN'),
                    string(credentialsId: 'tmdb', variable: 'API_KEY')
                ]) {
                    sh '''
                        echo "$DOCKER_TOKEN" | docker login -u bodkekarbalaji95 --password-stdin
                        docker build --build-arg TMDB_V3_API_KEY=$API_KEY -t netflix .
                        docker tag netflix bodkekarbalaji95/netflix:latest
                        docker push bodkekarbalaji95/netflix:latest
                    '''
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
                sh 'docker run -d --name netflix -p 8081:80 bodkekarbalaji95/netflix:latest || true'
            }
        }
    }
}
