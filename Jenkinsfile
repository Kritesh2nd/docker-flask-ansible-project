pipeline {
    agent any

    parameters {
        string(
            name: 'ANSIBLE_CONTROLLER_IP',
            defaultValue: '172.31.87.63',
            description: 'IP address of the Ansible controller'
        )

        string(
            name: 'ANSIBLE_CONTROLLER_USER',
            defaultValue: 'ubuntu',
            description: 'SSH user for Ansible controller'
        )
    }

    environment {
        APP_NAME          = 'simple-docker-flask-app'
        DOCKER_IMAGE_NAME = 'moudle8848/ansible-deploy'
    }

    stages {

        // CHECKOUT
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'

                checkout scm

                script {
                    def branch = sh(
                        script: 'git rev-parse --abbrev-ref HEAD',
                        returnStdout: true
                    ).trim()

                    // Jenkins may return HEAD in detached mode
                    if (branch == 'HEAD') {
                        branch = 'main'
                    }

                    branch = branch.replaceAll('/', '-')

                    def commit = sh(
                        script: 'git rev-parse --short=7 HEAD',
                        returnStdout: true
                    ).trim()

                    env.IMAGE_TAG = "${branch}-${commit}"

                    echo "Branch: ${branch}"
                    echo "Commit: ${commit}"
                    echo "Docker image: ${DOCKER_IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }

        // VALIDATE DOCKER COMPOSE
        stage('Validate Configuration') {
            steps {
                echo 'Validating Docker Compose configuration...'

                sh '''
                    docker compose config
                '''
            }
        }

        // BUILD DOCKER IMAGE
        stage('Build Docker Image') {
            steps {
                echo "Building Docker image ${DOCKER_IMAGE_NAME}:${IMAGE_TAG}..."

                sh """
                    docker build \
                        -t ${DOCKER_IMAGE_NAME}:${IMAGE_TAG} \
                        .
                """

                echo 'Built Docker images:'

                sh """
                    docker images ${DOCKER_IMAGE_NAME}
                """
            }
        }

        // TEST DEPLOYMENT
        stage('Test Deployment') {
            steps {
                echo 'Starting application stack with Docker Compose...'

                sh '''
                    WEB_PORT=8081 docker compose up -d
                '''

                echo 'Waiting for containers to become healthy...'

                sh '''
                    for i in $(seq 1 12); do

                        CONTAINER_ID=$(docker compose ps -q web)

                        if [ -z "$CONTAINER_ID" ]; then
                            echo "Web container has not been created yet."
                            sleep 5
                            continue
                        fi

                        STATUS=$(docker inspect \
                            --format="{{.State.Health.Status}}" \
                            "$CONTAINER_ID" 2>/dev/null || true)

                        echo "Web container health: $STATUS"

                        if [ "$STATUS" = "healthy" ]; then
                            echo "Web container is healthy."
                            exit 0
                        fi

                        sleep 5
                    done

                    echo "Web container failed to become healthy."

                    echo "Docker Compose status:"
                    docker compose ps

                    echo "Docker Compose logs:"
                    docker compose logs

                    exit 1
                '''

                echo 'Docker Compose status:'

                sh '''
                    docker compose ps
                '''
            }
        }

        // PUSH IMAGE TO DOCKER HUB        
        stage('Push Docker Image') {
            steps {
                echo 'Logging into Docker Hub...'

                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-creds',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {

                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login \
                            --username "$DOCKER_USERNAME" \
                            --password-stdin
                    '''

                    echo "Pushing ${DOCKER_IMAGE_NAME}:${IMAGE_TAG}..."

                    sh """
                        docker push ${DOCKER_IMAGE_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }

        // DEPLOY THROUGH ANSIBLE        
        stage('Deploy via Ansible') {
            steps {

                echo "Deploying ${DOCKER_IMAGE_NAME}:${IMAGE_TAG} through Ansible..."

                sshagent(credentials: ['ansible-ssh-key']) {

                    sh """
                        ssh -o StrictHostKeyChecking=no \
                            ${params.ANSIBLE_CONTROLLER_USER}@${params.ANSIBLE_CONTROLLER_IP} '
                                cd simple-docker-flask-app-ansible &&
                                ansible-playbook deploy.yml \
                                --extra-vars "docker_image=${DOCKER_IMAGE_NAME}" \
                                --extra-vars "image_tag=${IMAGE_TAG}"
                            '
                    """
                }
            }
        }
    }

    // POST ACTIONS
    post {

        always {
            echo 'Cleaning up Docker resources...'

            sh '''
                docker logout || true
                docker compose down -v --remove-orphans || true
            '''

            cleanWs()
        }

        success {
            echo 'Pipeline completed successfully!'
            echo "Deployed image: ${DOCKER_IMAGE_NAME}:${IMAGE_TAG}"
        }

        failure {
            echo 'Pipeline failed!'
        }
    }
}