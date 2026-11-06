pipeline {
    agent any

    environment {
        DOCKERHUB_USER = 'sinchanadas'
        APP_IP         = '13.126.139.252'
        APP_SERVER     = "ubuntu@${APP_IP}"
        APP_DIR        = '/home/ubuntu/CRUD-Node.js-React.js-MySQL'
        IMAGE_TAG      = "${BUILD_NUMBER}"
    }

    options {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log -1 --oneline'
            }
        }

        stage('Build Images') {
            steps {
                sh 'docker build -t $DOCKERHUB_USER/student-backend:$IMAGE_TAG ./server'
                sh 'docker build -t $DOCKERHUB_USER/student-frontend:$IMAGE_TAG ./client'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_PASS')]) {
                    sh '''
                        echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                        docker push $DOCKERHUB_USER/student-backend:$IMAGE_TAG
                        docker push $DOCKERHUB_USER/student-frontend:$IMAGE_TAG
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent(credentials: ['app-server-ssh']) {
                    sh '''
                        SSH_OPTS="-o StrictHostKeyChecking=no"
                        scp $SSH_OPTS docker-compose.prod.yml $APP_SERVER:$APP_DIR/
                        scp $SSH_OPTS db/init.sql $APP_SERVER:$APP_DIR/db/
                        ssh $SSH_OPTS $APP_SERVER "cd $APP_DIR && \
                          export DOCKERHUB_USER=$DOCKERHUB_USER IMAGE_TAG=$IMAGE_TAG && \
                          docker compose -f docker-compose.prod.yml pull && \
                          docker compose -f docker-compose.prod.yml up -d && \
                          docker image prune -af"
                    '''
                }
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    for i in $(seq 1 10); do
                        if curl -fs http://$APP_IP/api/students > /dev/null; then
                            echo "App is healthy - build $IMAGE_TAG is live at http://$APP_IP"
                            exit 0
                        fi
                        echo "Waiting for app ($i/10)..."; sleep 6
                    done
                    echo "Health check failed"; exit 1
                '''
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
            sh 'docker rmi $DOCKERHUB_USER/student-backend:$IMAGE_TAG $DOCKERHUB_USER/student-frontend:$IMAGE_TAG || true'
        }
    }
}