pipeline {
    agent any
    environment {
        DOCKER_REGISTRY_CREDENTIALS = 'dockerhub-credentials'
        HELM_CHART_PATH = './helm/jenkins-exam'
        KUBECONFIG = credentials("config")
        BRANCH_NAME = "${env.GIT_BRANCH?.replaceFirst(/^origin\//, '') ?: 'staging'}"
    }
    stages {
        stage('Clone Repository') {
            steps {
                git 'https://github.com/tangos974/ds-jenkins-exam'
            }
        }
        stage('Build and Push Docker Images') {
            steps {
                script {
                    def imageTag = (env.BRANCH_NAME == 'master') ? 'latest' : env.BRANCH_NAME
                    
                    // Build and push movie-service image
                    docker.withRegistry('', "${DOCKER_REGISTRY_CREDENTIALS}") {
                        def movieImage = docker.build("tanguyfremont/movie-service:${imageTag}", "./movie-service")
                        movieImage.push()
                    }
                    
                    // Build and push cast-service image
                    docker.withRegistry('', "${DOCKER_REGISTRY_CREDENTIALS}") {
                        def castImage = docker.build("tanguyfremont/cast-service:${imageTag}", "./cast-service")
                        castImage.push()
                    }
                }
            }
        }
        stage('Deploy to Kubernetes') {
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
                    // Deploy the app to the correct namespace
                    sh """
                        helm upgrade --install jenkinsexam ${HELM_CHART_PATH} \
                        --set movieService.image.tag=${env.BRANCH_NAME} \
                        --set castService.image.tag=${env.BRANCH_NAME} \
                        --namespace ${namespace} \
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
                    // Deploy entire application to production
                    sh """
                        helm upgrade --install jenkinsexam ${HELM_CHART_PATH} \
                        --set movieService.image.tag=latest \
                        --set castService.image.tag=latest \
                        --namespace prod \
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