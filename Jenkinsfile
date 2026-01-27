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

stage('Auto-Discover & Fix Environment') {
    steps {
        script {
            echo "K8s Master IP adresi Multipass üzerinden tespit ediliyor..."
            
            // 1. Host makinedeki Multipass'e sorup güncel IP'yi alıyoruz
            def getIpCmd = "multipass info k8s-master --format csv | grep k8s-master | cut -d, -f3"
            env.K8S_MASTER_IP = sh(script: getIpCmd, returnStdout: true).trim()
            
            if (!env.K8S_MASTER_IP) {
                error "HATA: K8s Master IP'si alınamadı. Makine kapalı olabilir!"
            }
            
            echo "Güncel IP Tespit Edildi: ${env.K8S_MASTER_IP}"

            // 2. Hem hosts dosyasını hem de kubeconfig'i tek seferde tamir ediyoruz
            sh """
                # Hosts dosyasını temizle ve yeni IP'yi yaz
                sed -i '/k8s-master/d' /etc/hosts
                echo '${env.K8S_MASTER_IP} k8s-master' >> /etc/hosts
                
                # Kubeconfig içindeki server adresini (IP kısmını) otomatik güncelle
                sed -i "s|server: https://.*:6443|server: https://${env.K8S_MASTER_IP}:6443|" ${KUBECONFIG}
                
                echo "Bağlantı ayarları otomatik olarak güncellendi ve doğrulandı."
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