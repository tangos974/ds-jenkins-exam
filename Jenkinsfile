pipeline {
    agent any

    environment {
        DOCKER_REGISTRY_CREDENTIALS = 'dockerhub-credentials'
        HELM_CHART_PATH = 'helm/jenkins-exam'
        KUBECONFIG = credentials("config")
        }

    stages {
        stage('Clone Repository') {
            steps {
                git 'https://github.com/tangos974/ds-jenkins-exam'
            }
        }

        stage('Build Movie Service Docker Image') {
            steps {
                script {
                    docker.build("tanguyfremont/movie-service:${env.BRANCH_NAME}", "./movie-service")
                }
            }
        }

        stage('Build Cast Service Docker Image') {
            steps {
                script {
                    docker.build("tanguyfremont/cast-service:${env.BRANCH_NAME}", "./cast-service")
                }
            }
        }

        stage('Push Movie Service Docker Image') {
            steps {
                script {
                    docker.withRegistry('', "${DOCKER_REGISTRY_CREDENTIALS}") {
                        docker.image("tanguyfremont/movie-service:${env.BRANCH_NAME}").push()
                    }
                }
            }
        }

        stage('Push Cast Service Docker Image') {
            steps {
                script {
                    docker.withRegistry('', "${DOCKER_REGISTRY_CREDENTIALS}") {
                        docker.image("tanguyfremont/cast-service:${env.BRANCH_NAME}").push()
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            when {
                expression {
                    return env.BRANCH_NAME != 'master' // Deploy to non-prod namespaces only if branch is not master
                }
            }
            steps {
                script {
                    def namespace = ''
                    if (env.BRANCH_NAME == 'dev') {
                        namespace = 'dev'
                    } else if (env.BRANCH_NAME == 'qa') {
                        namespace = 'qa'
                    } else if (env.BRANCH_NAME == 'staging') {
                        namespace = 'staging'
                    } else {
                        error("Unsupported branch for deployment: ${env.BRANCH_NAME}")
                    }

                    // Deploy movie-service
                    sh """
                        helm upgrade --install movie-service ${HELM_CHART_PATH} \
                        --set image.tag=${env.BRANCH_NAME} \
                        --namespace ${namespace} \
                        --set image.repository=tanguyfremont/movie-service \
                        --set service.name=movie-service \
                        --wait
                    """

                    // Deploy cast-service
                    sh """
                        helm upgrade --install cast-service ${HELM_CHART_PATH} \
                        --set image.tag=${env.BRANCH_NAME} \
                        --namespace ${namespace} \
                        --set image.repository=tanguyfremont/cast-service \
                        --set service.name=cast-service \
                        --wait
                    """
                }
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'master'  // Only allow on master branch
            }
            steps {
                input(message: 'Deploy to production?', ok: 'Deploy')  // Manual approval
                script {
                    // Deploy movie-service to production
                    sh """
                        helm upgrade --install movie-service ${HELM_CHART_PATH} \
                        --set image.tag=latest \
                        --namespace prod \
                        --set image.repository=tanguyfremont/movie-service \
                        --set service.name=movie-service \
                        --wait
                    """

                    // Deploy cast-service to production
                    sh """
                        helm upgrade --install cast-service ${HELM_CHART_PATH} \
                        --set image.tag=latest \
                        --namespace prod \
                        --set image.repository=tanguyfremont/cast-service \
                        --set service.name=cast-service \
                        --wait
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}