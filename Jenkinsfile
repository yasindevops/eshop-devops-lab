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
                # 1. Dosyanın yedeğini al (güvenlik için)
                cp ${KUBECONFIG} ${KUBECONFIG}.bak

                 # 2. Dosya içindeki her yerde ama her yerde 'k8s-master' gördüğün an IP'yi bas.
                    # 'g' (global) ve 'I' (ignore case) kullanarak her türlü varyasyonu yakalıyoruz.
                    sed -i "s/k8s-master/${env.K8S_MASTER_IP}/gI" ${KUBECONFIG}

                # 3. Dosyayı kontrol et (Loglarda neye dönüştüğünü görelim - Güvenlik uyarısı verebilir ama sorunu çözer)
                echo "--- GÜNCELLENEN CONFIG (İLK 10 SATIR) ---"
                head -n 10 ${KUBECONFIG}
                echo "----------------------------------------"
                """
            }
        }
    }
}


stage('Deploy to K3s') {
    steps {
        withCredentials([file(credentialsId: 'k3s-kubeconfig', variable: 'KUBECONFIG')]) {
            sh """
                # --validate=false ekleyerek DNS sorgusunu (openapi) devre dışı bırakıyoruz
                kubectl --kubeconfig=${KUBECONFIG} apply -f deployment.yaml --validate=false
    
                 # İmaj güncelleme
                kubectl --kubeconfig=${KUBECONFIG} set image deployment/eshop-web-app my-app=${REGISTRY}/${IMG_NAME}:${BUILD_NUMBER}
                """
        }
    }
}
    }
}