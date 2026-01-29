pipeline {
    agent any
    
    environment {
        // Altyapı artık sabit, değişkenler net
        REGISTRY    = "nexus-server.local:8083"
        K8S_MASTER  = "192.168.1.60"
        IMG_NAME    = "eshop-web"
    }

 stages {

stage('SonarQube Analysis') {
    steps {
        script {
            // Tools'tan scanner'ı çekiyoruz
            def scannerHome = tool 'SonarScanner' 
            
            // System ayarlarındaki 'SonarQube-Server' ismini kullanıyoruz
            // withSonarQubeEnv otomatik olarak gerekli ortam değişkenlerini (token vb.) içeri aktarır
            withSonarQubeEnv('SonarQube-Server') { 
sh """
    # Globalization hatasını bypass etmek için:
    export DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=1
    export PATH="\$PATH:/var/jenkins_home/.dotnet/tools"
    
    dotnet sonarscanner begin /k:"eshop-web-app" \
        /d:sonar.host.url="http://192.168.1.80:9000" \
        /d:sonar.token=sqa_59cde8276b79bae8e6a1e1f4b5837787ad19c15f
    
    dotnet build src/eShopOnWeb.sln --configuration Release
    
    dotnet sonarscanner end /d:sonar.token=sqa_59cde8276b79bae8e6a1e1f4b5837787ad19c15f
"""
            }
        }
    }
}

   
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