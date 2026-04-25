pipeline {
    agent any

    tools {
        jdk 'jdk21'
        nodejs 'node23'
    }

    environment {
        SONARQUBE_ENV = 'sq'
        DOCKER_IMAGE = "sujanvjijay/zomato"
        AWS_DEFAULT_REGION = 'us-east-1'
        RECIPIENTS = 'sujanvijay2311@gmail.com'
    }

    stages {

        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/rajeshtutta/zomato.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Build Application') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Run Tests') {
            steps {
                sh 'npm test || true'
            }
        }

        stage('Package Artifact') {
            steps {
                sh 'zip -r zomato-build.zip build/'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh '''
                    sonar-scanner \
                      -Dsonar.projectKey=zomato \
                      -Dsonar.sources=src \
                      -Dsonar.projectName=Zomato-App \
                      -Dsonar.projectVersion=${BUILD_NUMBER}
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Upload to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-cred',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh '''
                    curl -v -u $NEXUS_USER:$NEXUS_PASS \
                    --upload-file zomato-build.zip \
                    http://localhost:8081/repository/raw-hosted/zomato-build-${BUILD_NUMBER}.zip
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                docker build -t $DOCKER_IMAGE:${BUILD_NUMBER} .
                docker tag $DOCKER_IMAGE:${BUILD_NUMBER} $DOCKER_IMAGE:latest
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-cred',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh '''
                    echo $PASS | docker login -u $USER --password-stdin
                    docker push $DOCKER_IMAGE:${BUILD_NUMBER}
                    docker push $DOCKER_IMAGE:latest
                    docker logout
                    '''
                }
            }
        }

        stage('Install Helm') {
             steps {
                 sh '''
               curl -LO https://get.helm.sh/helm-v3.14.0-linux-amd64.tar.gz
               tar -zxvf helm-v3.14.0-linux-amd64.tar.gz
               mv linux-amd64/helm helm
               chmod +x helm
               '''
            }
        }
        
        stage('Verify Helm') {
    steps {
        sh './helm version'
    }
}

stage('Add Helm Repo') {
    steps {
        sh './helm repo add prometheus-community https://prometheus-community.github.io/helm-charts'
        sh './helm repo update'
    }
}

stage('Install Monitoring Stack') {
    steps {
        sh './helm install monitoring prometheus-community/kube-prometheus-stack'
    }
}

        stage('Deploy to EKS') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                    sh '''
                    aws eks update-kubeconfig --region us-east-1 --name mycluster
                    kubectl apply -f deployment.yml
                    kubectl apply -f service.yml
                    '''
                }
            }
        }
    }

    post {
        success {
            emailext(
                subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Build succeeded!\n\nCheck: ${env.BUILD_URL}",
                to: "${RECIPIENTS}"
            )
        }

        failure {
            emailext(
                subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Build failed!\n\nCheck: ${env.BUILD_URL}",
                to: "${RECIPIENTS}"
            )
        }

        always {
            archiveArtifacts artifacts: 'zomato-build.zip', fingerprint: true
        }
    }
}
