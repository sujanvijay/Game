pipeline {
    agent any



    environment {
        DOCKER_USER = "surya8442"
        IMAGE_NAME = "sliding-block-puzzle-game"
        IMAGE_TAG = "v1"
        KUBECONFIG = '/var/lib/jenkins/.kube/config'
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
                        sonar-scanner \
                        -Dsonar.projectKey=game \
                        -Dsonar.sources=src \
                        -Dsonar.projectName=game-App \
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

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG .'
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'Docker_cred', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh '''
                        echo $PASS | docker login -u $USER --password-stdin
                        docker push $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    aws eks update-kubeconfig --region ap-south-1 --name mycluster
                    kubectl get nodes
                    kubectl apply -f deployment.yml
                    kubectl apply -f service.yml
                '''
            }
        }
    }

    post {

        success {
            mail to: 'suryakandipalli@gmail.com',
            subject: "SUCCESS: NodeJS Pipeline - ${JOB_NAME}",
            body: """
Hi Surya,

Pipeline SUCCESSFUL 🚀

Project: Zomato-App (NodeJS)
Build Number: ${BUILD_NUMBER}
Docker Image: ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}

All stages completed successfully:
✔ Git Checkout
✔ Build
✔ SonarQube Passed
✔ Docker Push
✔ Kubernetes Deploy

Regards,
Jenkins CI/CD
"""
        }

        failure {
            mail to: 'suryakandipalli@gmail.com',
            subject: "FAILED: NodeJS Pipeline - ${JOB_NAME}",
            body: """
Hi Surya,

Pipeline FAILED ❌

Project: Zomato-App (NodeJS)
Build Number: ${BUILD_NUMBER}

Please check Jenkins logs for error details.

Regards,
Jenkins CI/CD
"""
        }

        always {
            echo 'Pipeline execution completed.'
        }
    }
}
