pipeline {
    agent any

    tools {
        jdk 'JDK17'
        maven 'Maven3'
    }

    environment {
        APP_NAME        = 'user-service'
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_IMAGE    = 'avnishkumar1998/foodflow-user-service'
        DOCKER_CREDS_ID = 'docker-registry-credentials'
        IMAGE_TAG       = "${env.BUILD_NUMBER}"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 1, unit: 'HOURS')
        timestamps()
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }

        stage('Compile & Test') {
            steps {
                echo 'Building and running unit tests...'
                sh '''
                    chmod +x mvnw || true
                    ./mvnw clean test -B
                '''
            }
            post {
                always {
                    junit testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: true
                }
            }
        }

        stage('Package Application') {
            steps {
                echo 'Packaging application JAR...'
                sh './mvnw package -DskipTests -B'
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image: ${DOCKER_IMAGE}:${IMAGE_TAG}..."
                sh """
                    docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} .
                    docker tag ${DOCKER_IMAGE}:${IMAGE_TAG} ${DOCKER_IMAGE}:latest
                """
            }
        }

        stage('Push to Registry') {
            when {
                anyOf {
                    branch 'main'
                    branch 'master'
                }
            }
            steps {
                echo 'Authenticating and pushing image to container registry...'
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDS_ID}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin ${DOCKER_REGISTRY}
                        docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                        docker push ${DOCKER_IMAGE}:latest
                    """
                }
            }
        }

        stage('Deploy (Kubernetes / OpenShift)') {
            when {
                anyOf {
                    branch 'main'
                    branch 'master'
                }
            }
            steps {
                echo 'Deploying to cluster...'
                // Example using kubectl / oc:
                // sh "kubectl set image deployment/user-service user-service=${DOCKER_IMAGE}:${IMAGE_TAG} -n foodflow"
                // sh "kubectl rollout status deployment/user-service -n foodflow"
                echo "Deployment triggered for image: ${DOCKER_IMAGE}:${IMAGE_TAG}"
            }
        }
    }

    post {
        always {
            cleanWs deleteDirs: true, notFailBuild: true
        }
        success {
            echo "Pipeline completed successfully for build #${env.BUILD_NUMBER}!"
        }
        failure {
            echo "Pipeline failed for build #${env.BUILD_NUMBER}. Please check logs."
        }
    }
}
