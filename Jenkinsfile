pipeline {
    agent any
    
    environment {
        REGISTRY    = "nexus-server.local:8083"
        K8S_MASTER  = "192.168.1.60"
        IMG_NAME    = "eshop-web"
    }

    stages {
        stage('SonarQube Analysis') {
            steps {
                // 'sonarqube-token' ID'li credential'ı güvenli bir şekilde çağırıyoruz
                withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                    script {
                        // Global Tool Configuration'daki ismi kullanıyoruz
                        def scannerHome = tool 'SonarScanner' 
                        
                        // System kısmındaki sunucu ismini kullanıyoruz
                        withSonarQubeEnv('SonarQube-Server') { 
                            sh """
                                export DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=1
                                export PATH="\$PATH:/var/jenkins_home/.dotnet/tools"

                                # 1. Önceki kalıntıları temizle
                                dotnet clean src/eShopOnWeb.sln

                                # 2. Analizi başlat (Exclusion ile)
                                dotnet-sonarscanner begin /k:"eshop-web-app" \
                                /d:sonar.host.url="http://192.168.1.80:9000" \
                                /d:sonar.token=${SONAR_TOKEN} \
                                /d:sonar.exclusions="**/_Imports.razor"
    
                                # 3. Yeniden build et
                                dotnet build src/eShopOnWeb.sln --configuration Release
    
                                # 4. Analizi bitir
                                dotnet-sonarscanner end /d:sonar.token=${SONAR_TOKEN}
                                """
                        }
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                sh "docker build -t ${REGISTRY}/${IMG_NAME}:${BUILD_NUMBER} -f src/src/Web/Dockerfile ."
                sh "docker push ${REGISTRY}/${IMG_NAME}:${BUILD_NUMBER}"
            }
        }

        stage('Trivy Security Scan') {
            steps {
                script {
                    // 'nexus-auth' ID'li credential'ı kullanıyoruz
                    withCredentials([usernamePassword(credentialsId: 'nexus-auth', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                        sh """
                            ssh -o StrictHostKeyChecking=no sonarqube@192.168.1.80 \
                            "TRIVY_USERNAME='${NEXUS_USER}' TRIVY_PASSWORD='${NEXUS_PASS}' \
                            trivy image --image-src remote \
                            --severity HIGH,CRITICAL \
                            --timeout 20m \
                            --exit-code 1 \
                            --insecure \
                            ${REGISTRY}/${IMG_NAME}:${BUILD_NUMBER}"
                        """
                    }
                }
            }
        }

        stage('Deploy to K3s') {
            steps {
                withCredentials([file(credentialsId: 'k3s-kubeconfig', variable: 'KUBECONFIG')]) {
                    sh """
                        kubectl --kubeconfig=${KUBECONFIG} --server=https://${K8S_MASTER}:6443 --insecure-skip-tls-verify apply -f deployment.yaml
                        kubectl --kubeconfig=${KUBECONFIG} --server=https://${K8S_MASTER}:6443 --insecure-skip-tls-verify set image deployment/eshop-web-app eshop-web=${REGISTRY}/${IMG_NAME}:${BUILD_NUMBER}
                    """
                }
            }
        }
    }
    
    // Her build sonrası disk temizliği 
    post {
        always {
            cleanWs()
        }
    }
}