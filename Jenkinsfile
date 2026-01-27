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

stage('Auto-Discover IP') {
    steps {
        withCredentials([file(credentialsId: 'k3s-kubeconfig', variable: 'KUBECONFIG')]) {
            script {
                // IP'yi dosyadan okuyoruz (Zaten bunu yapmıştık)
                env.K8S_MASTER_IP = readFile('/var/jenkins_home/k8s_ip.txt').trim()
                
                sh """
                    # KRİTİK NOKTA: Kubeconfig içindeki 'k8s-master' ismini 
                    # tamamen silip yerine direkt IP adresini yazıyoruz.
                    # Böylece kubectl artık DNS sorgusu (8.8.8.8) yapmaz.
                    sed -i "s|https://k8s-master:6443|https://${env.K8S_MASTER_IP}:6443|" ${KUBECONFIG}
                    sed -i "s|server: https://.*:6443|server: https://${env.K8S_MASTER_IP}:6443|" ${KUBECONFIG}
                    
                    echo "Kubeconfig artık isim yerine direkt IP (${env.K8S_MASTER_IP}) kullanıyor."
                """
            }
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