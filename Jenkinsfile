pipeline {
    agent any
    environment {
        REGISTRY = "nexus-server.local:8083"
        IMG_NAME = "eshop-web"
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    // Loglarındaki src/src/Web yapısına göre güncellendi
                    sh "docker build -t ${REGISTRY}/${IMG_NAME}:${BUILD_NUMBER} -f src/src/Web/Dockerfile ."
                    sh "docker tag ${REGISTRY}/${IMG_NAME}:${BUILD_NUMBER} ${REGISTRY}/${IMG_NAME}:latest"
                }
            }
        }

        stage('Push to Nexus') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'nexus-auth', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        sh "echo ${PASS} | docker login ${REGISTRY} -u ${USER} --password-stdin"
                        sh "docker push ${REGISTRY}/${IMG_NAME}:${BUILD_NUMBER}"
                    }
                }
            }
        }

stage('Fix DNS') {
    steps {
        script {
            // K8s Master'ın güncel IP'sini buraya yaz
            def k8sMasterIp = "172.17.167.89" 
            sh """
                # Eğer k8s-master kaydı yoksa ekle
                if ! grep -q "k8s-master" /etc/hosts; then
                    echo "${k8sMasterIp} k8s-master" >> /etc/hosts
                else
                    # Eğer varsa ama IP yanlışsa güncelle
                    sed -i "s/.*k8s-master/${k8sMasterIp} k8s-master/" /etc/hosts
                fi
            """
        }
    }
}


stage('Deploy to K3s') {
    steps {
        withCredentials([file(credentialsId: 'k3s-kubeconfig', variable: 'KUBECONFIG')]) {
            sh """
                # 1. Dosyayı uyguluyoruz
                kubectl --kubeconfig=${KUBECONFIG} apply -f deployment.yaml
                
                # 2. İmajı güncelliyoruz (Konteyner adını 'my-app' yaptık)
                kubectl --kubeconfig=${KUBECONFIG} set image deployment/eshop-web-app my-app=${REGISTRY}/${IMG_NAME}:${BUILD_NUMBER}
            """
        }
    }
}
    }
}