pipeline {
    agent any

    parameters {
        string(name: 'REGISTRY_URL', defaultValue: '192.168.0.191:30504', description: 'Docker registry URL')
        string(name: 'DOCKER_IMAGE', defaultValue: 'demo', description: 'Base name of the Docker image')
        booleanParam(name: 'BUILD_FLAG', defaultValue: false, description: 'Flag to control image build and push')
    }

    environment {
        REGISTRY_CREDENTIALS = 'harbor-cred'
        APP1_DIR = 'app1'
        APP2_DIR = 'test'
        BUILD_FLAG = false
    }

    triggers {
        pollSCM('H/5 * * * *') 
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    checkout scm
                }
            }
        }

        stage('Get Latest Tag') {
            steps {
                script {
                    sh 'git fetch --tags'
                    //latest tag
                    env.VERSION = sh(script: "git describe --tags --abbrev=0", returnStdout: true).trim()
                }
            }
        }

        stage('Build, Test and SonarQube Code Analysis') {
            parallel {
                stage('Build and Test App1') {
                    steps {
                        script {
                            def dockerTag = "${REGISTRY_URL}/${DOCKER_IMAGE}:${env.VERSION}"
                            buildAndTestImage(APP1_DIR, dockerTag)
                        }
                    }
                }

                stage('Build and Test App2') {
                    when {
                        changeset "**/${APP2_DIR}/**"
                    }
                    steps {
                        script {
                            def dockerTag = "${REGISTRY_URL}/${DOCKER_IMAGE}:${env.VERSION}"
                            buildAndTestImage(APP2_DIR, dockerTag)
                        }
                    }
                }

            }
        }

        stage('Push Images') {
            when {
                expression { return env.BUILD_FLAG == 'true' }
            }
            steps {
                script {
                    def dockerTag = "${REGISTRY_URL}/${DOCKER_IMAGE}:${env.VERSION}"
                    
                    echo "Pushing Docker Image: ${dockerTag}"
                    
                    withCredentials([usernamePassword(credentialsId: REGISTRY_CREDENTIALS, usernameVariable: 'DOCKER_REGISTRY_USERNAME', passwordVariable: 'DOCKER_REGISTRY_PASSWORD')]) {
                        sh """
                            echo "$DOCKER_REGISTRY_PASSWORD" | docker login -u $DOCKER_REGISTRY_USERNAME --password-stdin ${REGISTRY_URL}
                            docker push ${dockerTag}
                            docker tag ${dockerTag} ${REGISTRY_URL}/${DOCKER_IMAGE}:latest
                            docker push ${REGISTRY_URL}/${DOCKER_IMAGE}:latest
                            docker logout ${REGISTRY_URL}
                        """
                    }
                }
            }
        }
    }
}

def buildAndTestImage(String appDir, String dockerTag) {
    echo "Building Docker image for ${appDir}: ${dockerTag}"
    sh """
        docker build -t ${dockerTag} ${appDir}
    """
    env.BUILD_FLAG = 'true'
}
