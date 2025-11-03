pipeline {
    agent any

    environment {
        REGISTRY = "huytm1996/demo"
        NAMESPACE = "prod"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/huytm1996/blue-green-demo.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // luân phiên blue/green
                    def currentVersion = sh(script: "kubectl get svc app-service -n $NAMESPACE -o jsonpath='{.spec.selector.version}'", returnStdout: true).trim()
                    def newVersion = (currentVersion == 'blue') ? 'green' : 'blue'
                    env.NEW_VERSION = newVersion
                    sh "docker build -t $REGISTRY:${newVersion} ."
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh '''
                    echo "$PASS" | docker login -u "$USER" --password-stdin
                    docker push $REGISTRY:$NEW_VERSION
                    '''
                }
            }
        }

        stage('Deploy New Version') {
            steps {
                sh '''
                kubectl apply -f deployment-$NEW_VERSION.yaml -n $NAMESPACE
                '''
            }
        }

        stage('Smoke Test') {
            steps {
                sh '''
                sleep 10
                kubectl rollout status deployment/app-$NEW_VERSION -n $NAMESPACE
                '''
            }
        }

        stage('Switch Service') {
            steps {
                sh '''
                kubectl patch svc app-service -n $NAMESPACE -p '{"spec":{"selector":{"version":"'$NEW_VERSION'"}}}'
                '''
            }
        }

        stage('Cleanup Old Version') {
            steps {
                script {
                    def oldVersion = (env.NEW_VERSION == 'blue') ? 'green' : 'blue'
                    sh "kubectl delete deployment app-${oldVersion} -n $NAMESPACE || true"
                }
            }
        }
    }
}
