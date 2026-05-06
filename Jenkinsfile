pipeline {
    agent {
        kubernetes {
            yaml '''
            apiVersion: v1
            kind: Pod
            metadata:
              name: jenkins-agent
            spec:
              containers:
              - name: docker
                image: docker:29.4.1-cli-alpine3.23
                command:
                - cat
                tty: true
                volumeMounts:
                - mountPath: "/var/run/docker.sock"
                  name: docker-socket
              volumes:
              - name: docker-socket
                hostPath:
                  path: "/var/run/docker.sock"
            '''
        }
    }

    environment {
        DOCKER_IMAGE_NAME = "huisuz/beatbuddy-backend"
        DOCKER_CREDENTIALS_ID = "dockerhub-access"
    }

    stages {
        stage('Docker Image Build & Push') {
            steps {
                container('docker') {
                    script {
                        def buildNumber = "${env.BUILD_NUMBER}"

                        sh 'docker logout'

                        withCredentials([usernamePassword(
                            credentialsId: DOCKER_CREDENTIALS_ID,
                            usernameVariable: 'DOCKER_USERNAME',
                            passwordVariable: 'DOCKER_PASSWORD'
                        )]) {
                            sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'
                        }

                        withEnv(["DOCKER_IMAGE_VERSION=${buildNumber}"]) {
                            sh 'docker build --no-cache -t $DOCKER_IMAGE_NAME:$DOCKER_IMAGE_VERSION ./'
                            sh 'docker push $DOCKER_IMAGE_NAME:$DOCKER_IMAGE_VERSION'
                        }
                    }
                }
            }
        }

        stage('Trigger beatbuddy-k8s-manifests') {
            steps {
                script {
                    def buildNumber = "${env.BUILD_NUMBER}"
                    withEnv(["DOCKER_IMAGE_VERSION=${buildNumber}"]) {
                        build job: 'beatbuddy-k8s-manifests',
                            parameters: [
                                string(name: 'DOCKER_IMAGE_VERSION', value: "${DOCKER_IMAGE_VERSION}")
                            ],
                            wait: true
                    }
                }
            }
        }
    }
}