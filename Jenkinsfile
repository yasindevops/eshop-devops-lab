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
        script {
            echo "K8s Master IP adresi kontrol ediliyor..."
            
            // 1. Önce Multipass üzerinden güncel IP'yi almayı dene
            try {
                def getIpCmd = "multipass info k8s-master --format csv | grep k8s-master | cut -d, -f3"
                env.K8S_MASTER_IP = sh(script: getIpCmd, returnStdout: true).trim()
                
                // 2. Aldığı IP'yi dosyaya da yazsın ki bir sonraki seferde okuyabilsin
                sh "echo ${env.K8S_MASTER_IP} > /var/jenkins_home/k8s_ip.txt"
                echo "Güncel IP Multipass üzerinden çekildi: ${env.K8S_MASTER_IP}"
            } 
            catch (Exception e) {
                echo "Multipass'e erişilemedi, eski dosyadan okunmaya çalışılıyor..."
                if (fileExists('/var/jenkins_home/k8s_ip.txt')) {
                    env.K8S_MASTER_IP = readFile('/var/jenkins_home/k8s_ip.txt').trim()
                } else {
                    error "HATA: Ne Multipass'e ulaşılabildi ne de k8s_ip.txt dosyası bulundu!"
                }
            }
        }
    }
}

stage('Deploy to K3s') {
    steps {
        withCredentials([file(credentialsId: 'k3s-kubeconfig', variable: 'KUBECONFIG')]) {
            script {
                def masterIp = env.K8S_MASTER_IP
                
                sh """
                    # 1. Dosyayı uyguluyoruz 
                    kubectl --kubeconfig=${KUBECONFIG} --server=https://${masterIp}:6443 --insecure-skip-tls-verify apply -f deployment.yaml --validate=false
                    
                    # 2. İmajı güncelliyoruz (DÜZELTİLEN KISIM: my-app yerine eshop-web)
                    kubectl --kubeconfig=${KUBECONFIG} --server=https://${masterIp}:6443 --insecure-skip-tls-verify set image deployment/eshop-web-app eshop-web=${REGISTRY}/${IMG_NAME}:${BUILD_NUMBER}
                """
            }
        }
    }
}

    }
}