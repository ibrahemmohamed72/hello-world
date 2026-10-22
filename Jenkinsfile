pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
    }

    environment {
        IMAGE_NAME = 'ibrahemmohamed72/hello-world'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Image') {
            steps {
                sh '''
                    set -eu
                    docker build \
                        -t "$IMAGE_NAME:build-$BUILD_NUMBER" .
                '''
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    sh '''
                        set +x
                        set -eu

                        export DOCKER_CONFIG="$(mktemp -d)"
                        trap 'rm -rf "$DOCKER_CONFIG"' EXIT

                        printf '%s' "$DOCKER_TOKEN" |
                            docker login \
                                -u "$DOCKER_USER" \
                                --password-stdin

                        docker push \
                            "$IMAGE_NAME:build-$BUILD_NUMBER"
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withEnv([
                    'KUBECONFIG=/var/lib/jenkins/.kube/hello-world-config'
                ]) {
                    sh '''
                        set -eu

                        kubectl -n hello-world set image \
                            deployment/hello-world \
                            hello-world="$IMAGE_NAME:build-$BUILD_NUMBER"

                        kubectl -n hello-world rollout status \
                            deployment/hello-world \
                            --timeout=180s
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Build, push and deployment succeeded: ${env.IMAGE_NAME}:build-${env.BUILD_NUMBER}"
        }

        failure {
            echo 'Pipeline failed. Check Console Output for the failed stage.'
        }
    }
}
