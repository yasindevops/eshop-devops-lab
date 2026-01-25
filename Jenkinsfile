pipeline {
    agent any
    environment {
        REGISTRY = "nexus-server.local:8083"
        IMG_NAME = "eshop-web"
        NEXUS_CRED = "nexus-credentials" // Jenkins'e eklediğin ID
        KUBE_CRED  = "k3s-kubeconfig"    // Jenkins'e eklediğin ID
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm // GitHub'dan kodu çek
            }
        }
        stage('Docker Build') {
            steps {
                script {
                    // Paylaştığın resimdeki Dockerfile yolunu kullanıyoruz
                    sh "docker build -t ${REGISTRY}/${IMG_NAME}:${BUILD_NUMBER} -f src/Web/Dockerfile ."
                    sh "docker tag ${REGISTRY}/${IMG_NAME}:${BUILD_NUMBER} ${REGISTRY}/${IMG_NAME}:latest"
                }
            }
        }
        stage('Push to Nexus') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${NEXUS_CRED}", passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                    sh "echo ${PASS} | docker login ${REGISTRY} -u ${USER} --password-stdin"
                    sh "docker push ${REGISTRY}/${IMG_NAME}:${BUILD_NUMBER}"
                    sh "docker push ${REGISTRY}/${IMG_NAME}:latest"
                }
            }
        }
        stage('Deploy to K3s') {
            steps {
                withCredentials([file(credentialsId: "${KUBE_CRED}", variable: 'KUBECONFIG')]) {
                    sh """
                    kubectl --kubeconfig=${KUBECONFIG} apply -f deployment.yaml
                    kubectl --kubeconfig=${KUBECONFIG} set image deployment/eshop-web-app eshop-web-container=${REGISTRY}/${IMG_NAME}:${BUILD_NUMBER}
                    """
                }
            }
        }
    }
}