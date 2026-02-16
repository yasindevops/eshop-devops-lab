pipeline {
    agent any
    
    environment {
        REGISTRY    = "nexus-server.local:8083"
        IMG_NAME    = "eshop-web"
        K8S_MASTER  = "192.168.1.60"
        SEC_SERVER  = "192.168.1.80" 
    }

    stages {
        stage('SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                    script {
                        def scannerHome = tool 'SonarScanner' 
                        withSonarQubeEnv('SonarQube-Server') { 
                            sh """
                                export DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=1
                                export PATH="\$PATH:/var/jenkins_home/.dotnet/tools"
                                
                                dotnet-sonarscanner begin /k:"eshop-web-app" \
                                    /d:sonar.host.url="http://${SEC_SERVER}:9000" \
                                    /d:sonar.token=${SONAR_TOKEN} \
                                    /d:sonar.exclusions="**/_Imports.razor"
                                
                                dotnet build src/eShopOnWeb.sln --configuration Release
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
                    withCredentials([usernamePassword(credentialsId: 'nexus-auth', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                        sh """
                            # Tabloyu konsola bas
                            ssh -i /var/jenkins_home/.ssh/id_ed25519 -o StrictHostKeyChecking=no sonarqube@${SEC_SERVER} \
                            "TRIVY_USERNAME=${NEXUS_USER} TRIVY_PASSWORD=${NEXUS_PASS} \
                            trivy image --image-src remote --severity HIGH,CRITICAL --insecure ${REGISTRY}/${IMG_NAME}:${BUILD_NUMBER}"

                            # HTML Raporu üret
                            ssh -i /var/jenkins_home/.ssh/id_ed25519 -o StrictHostKeyChecking=no sonarqube@${SEC_SERVER} \
                            "TRIVY_USERNAME=${NEXUS_USER} TRIVY_PASSWORD=${NEXUS_PASS} \
                            trivy image --image-src remote --format template --template '@/usr/local/share/trivy/templates/html.tpl' \
                            -o /home/sonarqube/trivy-report.html --insecure ${REGISTRY}/${IMG_NAME}:${BUILD_NUMBER}"

                            # Raporu Jenkins'e çek
                            scp -i /var/jenkins_home/.ssh/id_ed25519 sonarqube@${SEC_SERVER}:/home/sonarqube/trivy-report.html .
                        """
                        archiveArtifacts artifacts: 'trivy-report.html', fingerprint: true
                    }
                }
            }
        }

 stage('Deploy to K3s') {
            steps {
                withCredentials([file(credentialsId: 'k3s-kubeconfig', variable: 'KUBECONFIG')]) {
                    sh """
                        # 1. Yeni imajı set et (Build numarasını zorla güncelle)
                        kubectl --kubeconfig=${KUBECONFIG} --server=https://${K8S_MASTER}:6443 --insecure-skip-tls-verify \
                        set image deployment/eshop-web-app eshop-web=${REGISTRY}/${IMG_NAME}:${BUILD_NUMBER}

                        # 2. Portu 80'e sabitle (ASPNETCORE_URLS ayarı)
                        kubectl --kubeconfig=${KUBECONFIG} --server=https://${K8S_MASTER}:6443 --insecure-skip-tls-verify \
                        set env deployment/eshop-web-app ASPNETCORE_URLS=http://+:80

                        # 3. ZORLAYICI ADIM: Podları yeni imajla yeniden başlatmaya zorla
                        # Bu komut, API server donup imajı tam set edemese bile süreci tetikler.
                        kubectl --kubeconfig=${KUBECONFIG} --server=https://${K8S_MASTER}:6443 --insecure-skip-tls-verify \
                        rollout restart deployment/eshop-web-app

                        # 4. Deployment'ın başarıyla tamamlanmasını bekle (Zaman aşımı 120sn)
                        kubectl --kubeconfig=${KUBECONFIG} --server=https://${K8S_MASTER}:6443 --insecure-skip-tls-verify \
                        rollout status deployment/eshop-web-app --timeout=120s

                        # 5. Gateway API statüsünü kontrol et
                        kubectl --kubeconfig=${KUBECONFIG} --server=https://${K8S_MASTER}:6443 --insecure-skip-tls-verify \
                        get gateway eshop-gateway
                    """
                }
            }
        }
    } 

    post {
        always {
            cleanWs()
        }
    }
}