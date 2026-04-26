pipeline {
    agent any

    tools {
        nodejs 'NodeJS18'
    }

    environment {
        DOCKER_USER = "surya8442"
        IMAGE_NAME = "sliding-block-puzzle-game"
        IMAGE_TAG = "v1"
        KUBECONFIG = '/var/lib/jenkins/.kube/config'

        NEXUS_URL = "http://43.204.30.42:8081/repository/puzzlegame"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Surya8442/Game.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sq') {
                    sh '''
                    /opt/sonar-scanner/bin/sonar-scanner \
                    -Dsonar.projectKey=game \
                    -Dsonar.sources=src \
                    -Dsonar.projectName=game-App \
                    -Dsonar.projectVersion=${BUILD_NUMBER}
                    '''
                }
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Package Artifact') {
            steps {
                sh '''
                if [ -d dist ]; then
                    tar -czf app-${BUILD_NUMBER}.tar.gz dist
                else
                    tar -czf app-${BUILD_NUMBER}.tar.gz .
                fi
                '''
            }
        }

        stage('Upload to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-credentials',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh '''
                    curl -v -u $NEXUS_USER:$NEXUS_PASS \
                    --upload-file app-${BUILD_NUMBER}.tar.gz \
                    $NEXUS_URL/app-${BUILD_NUMBER}.tar.gz
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG .'
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'Docker_cred',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh '''
                    echo $PASS | docker login -u $USER --password-stdin
                    docker push $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG
                    '''
                }
            }
        }

        stage('Install Helm') {
            steps {
                sh '''
                curl -LO https://get.helm.sh/helm-v3.14.0-linux-amd64.tar.gz
                tar -zxvf helm-v3.14.0-linux-amd64.tar.gz
                mv linux-amd64/helm ./helm
                chmod +x ./helm
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                aws eks update-kubeconfig --region ap-south-1 --name mycluster
                kubectl apply -f deployment.yml
                kubectl apply -f service.yml
                '''
            }
        }

        stage('Deploy Monitoring') {
            steps {
                sh '''
                ./helm repo add prometheus-community https://prometheus-community.github.io/helm-charts || true
                ./helm repo update

                ./helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
                --namespace monitoring --create-namespace
                '''
            }
        }

        stage('Expose Grafana') {
            steps {
                sh '''
                echo "Waiting for Grafana service..."
                sleep 30

                kubectl get svc -n monitoring

                kubectl patch svc monitoring-grafana \
                -n monitoring \
                -p '{"spec": {"type": "LoadBalancer"}}'
                '''
            }
        }
    }

    post {
        success {
            emailext(
                subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Build SUCCESS 🎉\n\nURL: ${env.BUILD_URL}",
                to: "${RECIPIENTS}"
            )
        }

        failure {
            emailext(
                subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Build FAILED ❌\n\nURL: ${env.BUILD_URL}",
                to: "${RECIPIENTS}"
            )
        }
    }
}
