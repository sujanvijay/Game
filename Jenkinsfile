pipeline {
    agent any

    tools {
        nodejs 'NodeJS18'   // Uses NodeJS configured in Jenkins
    }

    environment {
        DOCKER_USER = "sujanvijay"
        IMAGE_NAME = "sliding-block-puzzle-game"
        IMAGE_TAG = "${BUILD_NUMBER}"

        KUBECONFIG = '/var/lib/jenkins/.kube/config'

        NEXUS_URL = "http://13.222.158.174:8081/repository/nodejs-game/"
        RECIPIENTS = "sujanvijay2311@gmail.com"

        CLUSTER_NAME = "mycluster"
        PROJECT_NAME = "Sliding Puzzle Game"
    }

    stages {

        // ===============================
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/sujanvijay/Game.git'
            }
        }

        // ===============================
        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        // ===============================
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sq') {
                    script {
                        def scannerHome = tool 'sonar-scanner'

                        sh """
                        ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=game \
                        -Dsonar.sources=src \
                        -Dsonar.projectName=game-App \
                        -Dsonar.projectVersion=3
                        """
                    }
                }
            }
        }

        // ===============================
        stage('Build Application') {
            steps {
                sh 'npm run build'
            }
        }

        // ===============================
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

        // ===============================
        stage('Upload to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-credentials',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh '''
                    curl -u $NEXUS_USER:$NEXUS_PASS \
                    --upload-file app-${BUILD_NUMBER}.tar.gz \
                    $NEXUS_URL/app-${BUILD_NUMBER}.tar.gz
                    '''
                }
            }
        }

        // ===============================
        stage('Docker Build') {
            steps {
                sh '''
                docker build -t $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG .
                docker tag $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG $DOCKER_USER/$IMAGE_NAME:latest
                '''
            }
        }

        // ===============================
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
                    docker push $DOCKER_USER/$IMAGE_NAME:latest
                    '''
                }
            }
        }

        // ===============================
        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                aws eks update-kubeconfig --region us-east-1 --name $CLUSTER_NAME

                kubectl apply -f deployment.yml
                kubectl apply -f service.yml
                '''
            }
        }

        // ===============================
        stage('Install Helm (GLOBAL FIX)') {
            steps {
                sh '''
                if ! command -v helm &> /dev/null
                then
                    echo "Installing Helm globally..."

                    curl -LO https://get.helm.sh/helm-v3.14.0-linux-amd64.tar.gz
                    tar -zxvf helm-v3.14.0-linux-amd64.tar.gz
                    sudo mv linux-amd64/helm /usr/local/bin/helm
                fi

                helm version
                '''
            }
        }

        // ===============================
        stage('Deploy Monitoring (Prometheus + Grafana)') {
            steps {
                sh '''
                helm repo add prometheus-community https://prometheus-community.github.io/helm-charts || true
                helm repo update

                helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
                --namespace monitoring --create-namespace
                '''
            }
        }

        // ===============================
        stage('Expose Grafana') {
            steps {
                sh '''
                sleep 30

                kubectl patch svc monitoring-grafana \
                -n monitoring \
                -p '{"spec": {"type": "LoadBalancer"}}'
                '''
            }
        }

        // ===============================
        stage('Expose Prometheus') {
            steps {
                sh '''
                kubectl patch svc monitoring-kube-prometheus-prometheus \
                -n monitoring \
                -p '{"spec": {"type": "LoadBalancer"}}'
                '''
            }
        }
    }

    // ==========================================
    post {

        success {
            script {

                sleep 40

                def APP_URL = ""
                def GRAFANA_URL = ""
                def PROM_URL = ""

                // Retry logic (VERY IMPORTANT for AWS LB delay)
                for (int i = 0; i < 6; i++) {

                    APP_URL = sh(
                        script: "kubectl get svc puzzle-game-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' || true",
                        returnStdout: true
                    ).trim()

                    GRAFANA_URL = sh(
                        script: "kubectl get svc monitoring-grafana -n monitoring -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' || true",
                        returnStdout: true
                    ).trim()

                    PROM_URL = sh(
                        script: "kubectl get svc monitoring-kube-prometheus-prometheus -n monitoring -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' || true",
                        returnStdout: true
                    ).trim()

                    if (APP_URL && GRAFANA_URL && PROM_URL) {
                        break
                    }

                    sleep 20
                }

                def DOCKER_IMAGE = "${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}"

                // ================= EMAIL FIX =================
                emailext(
                    subject: "🚀 SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    mimeType: 'text/html',
                    from: "sujanvijay2311@gmail.com",
                    to: "${env.RECIPIENTS}",
                    body: "Deployment Successful"
                    <html>
                    <body>

                    <h2 style="color:green;">Deployment Successful</h2>

                    <p><b>Project:</b> ${PROJECT_NAME}</p>
                    <p><b>Cluster:</b> ${CLUSTER_NAME}</p>

                    <h3>Docker Image</h3>
                    <p>${DOCKER_IMAGE}</p>

                    <h3>Application</h3>
                    <a href="http://${APP_URL}">${APP_URL}</a>

                    <h3>Grafana</h3>
                    <a href="http://${GRAFANA_URL}">${GRAFANA_URL}</a>

                    <h3>Prometheus</h3>
                    <a href="http://${PROM_URL}:9090">${PROM_URL}</a>

                    <h3>Jenkins</h3>
                    <a href="${env.BUILD_URL}">Build Link</a>

                    </body>
                    </html>
                    """
                )
            }
        }

        failure {
            emailext(
                subject: "❌ FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                mimeType: 'text/html',
                to: "${env.RECIPIENTS}",
                body: """
                <html>
                <body>

                <h2 style="color:red;">Deployment Failed</h2>

                <p><b>Project:</b> Sliding Puzzle Game</p>
                <p><b>Cluster:</b> mycluster</p>

                <a href="${env.BUILD_URL}">View Logs</a>

                </body>
                </html>
                """
            )
        }

        always {
            archiveArtifacts artifacts: 'app-*.tar.gz', fingerprint: true, allowEmptyArchive: true
        }
    }
}
