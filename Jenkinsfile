pipeline {
    agent any

    tools {
        jdk 'jdk21'
        nodejs 'nodejs'
    }

    environment {
        IMAGE_NAME = 'zomato-app'
        DOCKER_IMAGE = "sridatta5157/zomato"
        AWS_REGION = "ap-southeast-2"
        CLUSTER_NAME = "mycluster"
        NEXUS_URL = 'http://54.206.60.159:8081/repository/raw_hosted'
        KUBECONFIG = "${env.HOME}/.kube/config"
        PATH = "/usr/local/bin:/var/lib/jenkins/bin:${env.PATH}"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scmGit(
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        credentialsId: 'git-cred',
                        url: 'https://github.com/sridattaga/Zomato_npm.git'
                    ]]
                )
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    node -v
                    npm -v
                    rm -rf node_modules package-lock.json
                    npm install
                '''
            }
        }

        stage('Build React App') {
            steps {
                sh '''
                    export CI=true
                    npm run build
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonar-cred', variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv('sonarscanner') {
                        script {
                            def scannerHome = tool name: 'sonarscanner'

                            sh """
                                ${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=zomato \
                                -Dsonar.projectName=Zomato-App \
                                -Dsonar.sources=. \
                                -Dsonar.projectVersion=${BUILD_NUMBER} \
                                -Dsonar.login=${SONAR_TOKEN} \
                                -Dsonar.token=${SONAR_TOKEN}
                            """
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Package Artifact') {
            steps {
                sh '''
                    zip -r zomato-build.zip build/
                '''
            }
        }

        stage('Upload to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-cred',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh """
                        curl -u $NEXUS_USER:$NEXUS_PASS \
                        --upload-file zomato-build.zip \
                        ${NEXUS_URL}/zomato-build-${BUILD_NUMBER}.zip
                    """
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh """
                    docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                    docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest
                """
            }
        }

        stage('Docker Login & Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-cred',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                        docker push ${DOCKER_IMAGE}:latest
                        docker logout
                    """
                }
            }
        }

        stage('Configure AWS EKS Access') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-cred'
                ]]) {
                    sh """
                        aws eks update-kubeconfig \
                            --region ${AWS_REGION} \
                            --name ${CLUSTER_NAME}

                        kubectl get nodes
                    """
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

        stage('Deploy Monitoring') {
            steps {
               sh '''
                ./helm repo add prometheus-community \
                https://prometheus-community.github.io/helm-charts || true

                ./helm repo update

                ./helm upgrade --install monitoring \
                prometheus-community/kube-prometheus-stack \
                --namespace monitoring \
                --create-namespace \
                --set grafana.service.type=LoadBalancer
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                    kubectl get pods -A
                '''
            }
        }
    }
}
