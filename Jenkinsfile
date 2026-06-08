pipeline {
    agent any

    environment {
        HARBOR_URL = "18.142.255.167:8081"
        HARBOR_PROJECT = "voting"
        HARBOR_CRED = credentials('harbor-admin')
        
        GIT_HELM_REPO = "https://github.com/BubbleTea1234/voting.git"
        GIT_HELM_CRED = credentials('github-token')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    GIT_HASH = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    env.BUILD_TAG = "build-${BUILD_NUMBER}-${GIT_HASH}"
                }
            }
        }

        stage('Build & Push Vote') {
            steps {
                dir('vote') {
                    sh """
                        docker build -t ${HARBOR_URL}/${HARBOR_PROJECT}/vote:${BUILD_TAG} .
                        docker login ${HARBOR_URL} -u ${HARBOR_CRED_USR} -p ${HARBOR_CRED_PSW}
                        docker push ${HARBOR_URL}/${HARBOR_PROJECT}/vote:${BUILD_TAG}
                    """
                }
            }
        }

        stage('Build & Push Result') {
            steps {
                dir('result') {
                    sh """
                        docker build -t ${HARBOR_URL}/${HARBOR_PROJECT}/result:${BUILD_TAG} .
                        docker push ${HARBOR_URL}/${HARBOR_PROJECT}/result:${BUILD_TAG}
                    """
                }
            }
        }

        stage('Build & Push Worker') {
            steps {
                dir('worker') {
                    sh """
                        docker build -t ${HARBOR_URL}/${HARBOR_PROJECT}/worker:${BUILD_TAG} .
                        docker push ${HARBOR_URL}/${HARBOR_PROJECT}/worker:${BUILD_TAG}
                    """
                }
            }
        }

        stage('Update Helm Chart') {
            steps {
                dir('helm-chart') {
                    sh """
                        git clone ${GIT_HELM_REPO} .
                        git config user.email "jenkins@voting.com"
                        git config user.name "Jenkins CI"
                        
                        # 更新 vote
                        sed -i 's|repository:.*|repository: ${HARBOR_URL}/${HARBOR_PROJECT}/vote|' vote/values.yaml
                        sed -i 's/tag:.*/tag: ${BUILD_TAG}/' vote/values.yaml
                        
                        # 更新 result
                        sed -i 's|repository:.*|repository: ${HARBOR_URL}/${HARBOR_PROJECT}/result|' result/values.yaml
                        sed -i 's/tag:.*/tag: ${BUILD_TAG}/' result/values.yaml
                        
                        # 更新 worker
                        sed -i 's|repository:.*|repository: ${HARBOR_URL}/${HARBOR_PROJECT}/worker|' worker/values.yaml
                        sed -i 's/tag:.*/tag: ${BUILD_TAG}/' worker/values.yaml
                        
                        git add -A
                        git commit -m "Auto update images to ${BUILD_TAG}"
                        git push https://${GIT_HELM_CRED_USR}:${GIT_HELM_CRED_PSW}@github.com/BubbleTea1234/voting.git main
                    """
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout ${HARBOR_URL}'
        }
    }
}
