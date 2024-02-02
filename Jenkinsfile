pipeline {
    agent any

    environment {
        KUBECONFIG = credentials('kube-config')
        DOCKER_ID = "soymhnz"
        DOCKER_TAG = "v.${BUILD_ID}.0"
        DOCKER_PASS = credentials("DOCKER_HUB_PASS")
    }

    stages {
        stage('Docker Build - movie-service') {
            steps {
                script {
                    def dockerImage
                    dockerImage = docker.build("soymhnz/jenkins_devops_exams-movie_service:${DOCKER_TAG}", "movie-service")
                }
            }
        }
        
        stage('Docker Build - cast-service') {
            steps {
                script {
                    def dockerImage
                    dockerImage = docker.build("soymhnz/jenkins_devops_exams-cast_service:${DOCKER_TAG}", "cast-service")
                }
            }
        }

        stage('Tests - movie-service') {
            steps {
                script {
                    sh'''
                    docker network create movie-network
                    docker run -d -t --name movie-db -v postgres_data_movie:/var/lib/postgresql/data/ -e POSTGRES_USER=movie_db_username -e POSTGRES_PASSWORD=movie_db_password -e POSTGRES_DB=movie_db_dev --network movie-network postgres:12.1-alpine
                    docker run -d -t --name movie-service-container -p 8001:8000 -v movie-service:/app/ -e DATABASE_URI=postgresql://movie_db_username:movie_db_password@movie_db/movie_db_dev -e CAST_SERVICE_HOST_URL=http://cast_service:8000/api/v1/casts/ --network movie-network $DOCKER_ID/jenkins_devops_exams-movie_service:$DOCKER_TAG
                    '''
                    sleep 10
                    sh '''
                    docker rm -f movie-service-container || true
                    docker rm -f movie-db || true
                    docker network rm movie-network || true
                    '''
                }
            }
        }

        stage('Tests - cast-service') {
            steps {
                script {
                    sh'''
                    docker network create cast-network
                    docker run -d -t --name cast-db -v postgres_data_cast:/var/lib/postgresql/data/ -e POSTGRES_USER=cast_db_username -e POSTGRES_PASSWORD=cast_db_password -e POSTGRES_DB=cast_db_dev --network cast-network postgres:12.1-alpine
                    docker run -d -t --name cast-service-container -p 8002:8000 -v cast-service:/app/ -e DATABASE_URI=postgresql://cast_db_username:cast_db_password@cast_db/cast_db_dev --network cast-network $DOCKER_ID/jenkins_devops_exams-cast_service:$DOCKER_TAG
                    '''
                    sleep 10
                    sh '''
                    docker rm -f cast-service-container || true
                    docker rm -f cast-db || true
                    docker network rm cast-network || true
                    '''
                }
            }
        }

        stage('Docker Push - movie-service') {
            steps {
                script {
                    sh '''
                    docker login -u $DOCKER_ID -p $DOCKER_PASS
                    docker push $DOCKER_ID/jenkins_devops_exams-movie_service:$DOCKER_TAG
                    '''
                }
            }
        }
        
        stage('Docker Push - cast-service') {
            steps {
                script {
                    sh '''
                    docker login -u $DOCKER_ID -p $DOCKER_PASS
                    docker push $DOCKER_ID/jenkins_devops_exams-cast_service:$DOCKER_TAG
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes - Dev') {
            steps {
                script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    ls
                    cat $KUBECONFIG > .kube/config
                    cp movie_service_chart/values.yaml values.yml
                    cat values.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                    helm upgrade --install movie-service movie_service_chart --namespace dev -f values.yml
                    '''
                    sleep 5
                    sh '''
                    cp cast_service_chart/values.yaml values.yml
                    cat values.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                    helm upgrade --install cast-service cast_service_chart --namespace dev -f values.yml
                    '''
                    sleep 5
                    sh '''
                    cp nginx-chart/values.yaml values.yml
                    cat values.yml
                    helm upgrade --install nginx nginx-chart --namespace dev -f values.yml
                    '''
                }
            }
        }
    
        stage('Deploy to Kubernetes - QA') {
            steps {
                script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    ls
                    cat $KUBECONFIG > .kube/config
                    cp movie_service_chart/values.yaml values.yml
                    cat values.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                    helm upgrade --install movie-service movie_service_chart --namespace qa -f values.yml
                    '''
                    sleep 5
                    sh '''
                    cp cast_service_chart/values.yaml values.yml
                    cat values.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                    helm upgrade --install cast-service cast_service_chart --namespace qa -f values.yml
                    '''
                    sleep 5
                    sh '''
                    cp nginx-chart/values.yaml values.yml
                    cat values.yml
                    helm upgrade --install nginx nginx-chart --namespace qa -f values.yml
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes - Stagging') {
            steps {
                script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    ls
                    cat $KUBECONFIG > .kube/config
                    cp movie_service_chart/values.yaml values.yml
                    cat values.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                    helm upgrade --install movie-service movie_service_chart --namespace stagging -f values.yml
                    '''
                    sleep 5
                    sh '''
                    cp cast_service_chart/values.yaml values.yml
                    cat values.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                    helm upgrade --install cast-service cast_service_chart --namespace stagging -f values.yml
                    '''
                    sleep 5
                    sh '''
                    cp nginx-chart/values.yaml values.yml
                    cat values.yml
                    helm upgrade --install nginx nginx-chart --namespace stagging -f values.yml
                    '''
                }
            }
        }

        stage('Deploiement en prod'){
            steps {
                // Create an Approval Button with a timeout of 15minutes.
                // this require a manuel validation in order to deploy on production environment
                        timeout(time: 15, unit: "MINUTES") {
                            input message: 'Do you want to deploy in production ?', ok: 'Yes'
                        }

                    script {
                    sh '''
                    cp cast_service_chart/values.yaml values.yml
                    cat values.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                    helm upgrade --install cast-service cast_service_chart --namespace prod -f values.yml
                    '''
                    sleep 5
                    sh '''
                    cp movie_service_chart/values.yaml values.yml
                    cat values.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                    helm upgrade --install movie-service movie_service_chart --namespace prod -f values.yml
                    '''
                    sleep 5
                    sh '''
                    cp nginx-chart/values.yaml values.yml
                    cat values.yml
                    helm upgrade --install nginx nginx-chart --namespace prod -f values.yml
                    '''
                    }
                }

            }

        }

    post {
        always {
            echo 'Pipeline execution complete.'
        }
    }
}
