pipeline {
    agent any

    tools {
        jdk 'jdk21'
        nodejs 'nodejs'
    }

    environment {
        DOCKER_IMAGE = "kalyansagar5/zomato"
        AWS_REGION   = "us-east-1"
        CLUSTER_NAME = "mycluster"
        RECIPIENTS   = "ravee2288@gmail.com"
        NEXUS_URL    = "http://100.54.40.237:8081/repository/raw_hosted"
    }

    stages {

        stage('Clone Repo') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/kalyansagar5/Zomato_npm.git',
                    credentialsId: 'Git_cred'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Build App') {
            steps {
                sh 'npm run build || true'
            }
        }

        stage('Test') {
            steps {
                sh 'npm test || true'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonarqube_cred', variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv('sonarscanner') {
                        sh '''
                        sonar-scanner \
                          -Dsonar.projectKey=zomato \
                          -Dsonar.sources=. \
                          -Dsonar.projectName=Zomato-App \
                          -Dsonar.projectVersion=${BUILD_NUMBER} \
                          -Dsonar.login=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 3, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Package Artifact') {
            steps {
                sh '''
                if [ -d build ]; then
                  zip -r zomato-build.zip build/
                else
                  echo "Build folder not found!" && exit 1
                fi
                '''
            }
        }

        stage('Upload to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus_cred',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh '''
                    curl -u $NEXUS_USER:$NEXUS_PASS \
                    --upload-file zomato-build.zip \
                    $NEXUS_URL/zomato-build-${BUILD_NUMBER}.zip
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                docker build -t $DOCKER_IMAGE:${BUILD_NUMBER} .
                docker tag $DOCKER_IMAGE:${BUILD_NUMBER} $DOCKER_IMAGE:latest
                '''
            }
        }

        stage('Docker Login & Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker_cred',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                    docker push $DOCKER_IMAGE:${BUILD_NUMBER}
                    docker push $DOCKER_IMAGE:latest
                    docker logout
                    '''
                }
            }
        }

        stage('Install Helm (if not exists)') {
            steps {
                sh '''
                if ! command -v helm &> /dev/null; then
                  curl -LO https://get.helm.sh/helm-v3.14.0-linux-amd64.tar.gz
                  tar -zxvf helm-v3.14.0-linux-amd64.tar.gz
                  mv linux-amd64/helm /usr/local/bin/helm
                  chmod +x /usr/local/bin/helm
                fi
                '''
            }
        }

        stage('Deploy Monitoring') {
            steps {
                sh '''
                helm repo add prometheus-community https://prometheus-community.github.io/helm-charts || true
                helm repo update

                helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
                --namespace monitoring --create-namespace
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws_cred'
                ]]) {
                    sh '''
                    export AWS_DEFAULT_REGION=$AWS_REGION

                    aws eks update-kubeconfig \
                        --region $AWS_REGION \
                        --name $CLUSTER_NAME

                    kubectl get nodes

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

        always {
            archiveArtifacts artifacts: 'zomato-build.zip', fingerprint: true
            cleanWs()
        }
    }
}
