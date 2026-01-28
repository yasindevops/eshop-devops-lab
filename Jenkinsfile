pipeline {
    agent any
    
    environment {
        // Altyapı artık sabit, değişkenler net
        REGISTRY    = "nexus-server.local:8083"
        K8S_MASTER  = "192.168.1.60"
        IMG_NAME    = "eshop-web"
    }

    stages {
        stage('Docker Build & Push') {
            steps {
                // Dockerfile yolu ve imaj etiketleme
                sh "docker build -t ${REGISTRY}/${IMG_NAME}:${BUILD_NUMBER} -f src/src/Web/Dockerfile ."
                sh "docker push ${REGISTRY}/${IMG_NAME}:${BUILD_NUMBER}"
            }
        }

        stage('Deploy to K3s') {
            steps {
                // Kubeconfig dosyasını Jenkins Credentials'tan alıyoruz
                withCredentials([file(credentialsId: 'k3s-kubeconfig', variable: 'KUBECONFIG')]) {
                    sh """
                        # Kubernetes'e yeni imajı ve deployment dosyasını gönder
                        kubectl --kubeconfig=${KUBECONFIG} --server=https://${K8S_MASTER}:6443 --insecure-skip-tls-verify apply -f deployment.yaml
                        
                        kubectl --kubeconfig=${KUBECONFIG} --server=https://${K8S_MASTER}:6443 --insecure-skip-tls-verify set image deployment/eshop-web-app eshop-web=${REGISTRY}/${IMG_NAME}:${BUILD_NUMBER}
                    """
                }
            }
        }
    }
}