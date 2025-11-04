pipeline {
    agent any

    tools {
        maven 'Maven3'
        jdk 'Java21'
    }

        environment {
        APP_NAME = "devops-03-pipeline-aws-gitops"
        RELEASE = "1.0"
        DOCKER_USER = "sadikgok"
        DOCKER_ID_LOGIN = 'dockerhub-sadikgok'
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}.${BUILD_NUMBER}"
        //JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN")

        // Trivy rapor dosyalarının dinamik isimleri
        TRIVY_JSON_REPORT = "trivy-report-${IMAGE_TAG}.json"
        TRIVY_HTML_TEMPLATE = "html.tpl"
        TRIVY_HTML_REPORT = "trivy-report-${IMAGE_TAG}.html"
    }

    stages {

       stage('SCM GitHub') {
            steps {
                checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[url: 'https://github.com/sadikgok/devops-03-pipeline-aws-gitops']])
            }
        }

         stage('Test Maven') {
            steps {
                script {
                    if (isUnix()) {
                        // Linux or MacOS
                        sh 'mvn test'
                    } else {
                        bat 'mvn test'  // Windows
                    }
                }
            }
        }
        stage('Build Maven') {
            steps {
                script {
                    if (isUnix()) {
                        sh 'mvn clean install'
                    } else {
                        bat 'mvn clean install'
                    }
                }
            }
        }

       stage("SonarQube Analysis") {
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'SonarTokenForJenkins') {
                        if (isUnix()) {
                            // Linux or MacOS
                            sh "mvn sonar:sonar"
                        } else {
                            bat 'mvn sonar:sonar'  // Windows
                        }
                    }
                }
            }
        }


       stage("Quality Gate"){
           steps {
               script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'SonarTokenForJenkins'
                }
            }
        }
        
 // 1. AŞAMA: Dosya Sistemi Taraması
        stage("Trivy File System Scan") {
            steps {
                script {
                    // Host'taki mevcut WORKSPACE dizinini (proje dosyaları)
                    // Trivy konteyneri içindeki /scan dizinine mount ediyoruz.
                    // Böylece trivy, proje dosyalarını /scan dizininde görebilir.
                    def trivy_fs_command = "docker run --rm " +
                                            "-v ${WORKSPACE}:/scan " +
                                            "aquasec/trivy fs /scan --no-progress > trivyfs.txt"

                    echo "Trivy Dosya Sistemi Taraması Başlatılıyor..."

                    if (isUnix()) {
                        sh trivy_fs_command
                    } else {
                        // Windows agent kullanılıyorsa (bat komutu ve path formatı değişir)
                        // Windows'ta volume mapping karmaşık olabileceğinden, bu kısım dikkatli test edilmelidir.
                        bat trivy_fs_command.replace('/scan', 'C:/scan') // Örnek path düzeltmesi
                    }
                    echo "Trivy Dosya Sistemi Taraması Tamamlandı."
                }
            }
        }

        // 2. AŞAMA: Docker Oluşturma ve Yayınlama
        stage('Docker Build & Push to DockerHub') {
            steps {
                script {
                    docker.withRegistry('', DOCKER_ID_LOGIN) {
                        def docker_image = docker.build "${IMAGE_NAME}"
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push("latest")
                    }
                }
            }
        }

// 3. AŞAMA: Trivy İmaj Taraması ve Güvenlik Zorunluluğu
        stage("Trivy Image Scan - Security Gate"){
            steps{
                script {
                    def imageToScan = "${IMAGE_NAME}:${IMAGE_TAG}" 
                    
                    // Önemli: Tarama ve raporu WORKSPACE içine yazabilmek için 
                    // hem docker.sock'ı hem de WORKSPACE'i mount ediyoruz.
                    def trivy_image_command = "docker run --rm " +
                                              "-v /var/run/docker.sock:/var/run/docker.sock " +
                                              "-v ${WORKSPACE}:/report " +
                                              "aquasec/trivy image " +
                                              "--exit-code 0 " +
                                              "--severity CRITICAL,HIGH " +
                                              "--format json " +
                                              "--output /report/${TRIVY_JSON_REPORT} " +
                                              "${imageToScan}"
                    
                    echo "Taranacak İmaj: ${imageToScan}"
                    
                    if (isUnix()) {
                        sh trivy_image_command
                    } else {
                        // Windows için bat komutu ve path düzenlemesi
                        bat trivy_image_command.replace('/report', 'C:/report')
                    }
                }
            }
        }
        
    stage('Download Trivy Template') {
    steps {
        sh 'curl -L https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl -o html.tpl'
    }
}


        // 4. AŞAMA: HTML Raporu Oluşturma
        stage("Trivy Image Scan - JSON + HTML") {
            steps {
                script {
                    def imageToScan = "${IMAGE_NAME}:${IMAGE_TAG}"
                    echo "Taranacak imaj: ${imageToScan}"

                    // Trivy'nin yazma izinlerine takılmaması için
                    sh "chmod 777 ${WORKSPACE} || true"

                    // JSON formatlı rapor (güvenlik için)
                    sh """
                        docker run --rm \
                            -v /var/run/docker.sock:/var/run/docker.sock \
                            -v ${WORKSPACE}:/report \
                            aquasec/trivy image ${imageToScan} \
                            --format json \
                            --output /report/${TRIVY_JSON_REPORT}
                    """

                    // HTML formatlı rapor (görselleştirme için)
                    sh """
                        docker run --rm \
                            -v /var/run/docker.sock:/var/run/docker.sock \
                            -v ${WORKSPACE}:/report \
                            aquasec/trivy image ${imageToScan} \
                            --format template \
                            --template "@html" \
                            --output /report/${TRIVY_HTML_REPORT}
                    """

                    echo "JSON ve HTML raporlar oluşturuldu."
                    sh "ls -lh ${WORKSPACE} | grep trivy-report || true"
                }
            }
    }




/*
        stage('Docker Image') {
            steps {
            //    sh 'docker build  -t mimaraslan/devops-application:latest   .'
                bat 'docker build  -t mimaraslan/devops-application:latest   .'
            }
        }


        stage('Docker Image To DockerHub') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'dockerhub', variable: 'dockerhub')]) {

                        if (isUnix()) {
                             sh 'docker login -u mimaraslan -p %dockerhub%'
                             sh 'docker push mimaraslan/devops-application:latest'
                          } else {
                             bat 'docker login -u mimaraslan -p %dockerhub%'
                             bat 'docker push mimaraslan/devops-application:latest'
                         }
                    }
                }
            }
        }
    

        stage('Deploy Kubernetes') {
            steps {
            script {
                    kubernetesDeploy (configs: 'deployment-service.yaml', kubeconfigId: 'kubernetes')
                }
            }
        }


        stage('Docker Image to Clean') {
            steps {
              
                   //  sh 'docker image prune -f'
                     bat 'docker image prune -f'
               
            }
        }
*/

    }

    // POST BLOK: Pipeline başarısız olsa bile raporu yayımla
    post {
        always {
            echo "Trivy has been published report..."
            
            // HTML Publisher Plugin kullanımı
            publishHTML(
                target: [
                    allowMissing         : false, 
                    alwaysLinkToLastBuild: true,  
                    keepAll              : true,  
                    reportDir            : "${WORKSPACE}",
                    reportFiles          : "${TRIVY_HTML_REPORT}",
                    reportName           : "Trivy Security Report - ${IMAGE_TAG}"
                ]
            )
        }

        success {
        echo "Pipeline başarılı. Docker imajları temizleniyor..."
        
        script {
            def REPO_NAME = "${IMAGE_NAME}"
            
         sh """
                echo "Eski imajlar için temizlik başlatılıyor (Son 3 imaj korunacak)..."
                
                # Yeni yaklaşım: Hem ID'yi hem de oluşturulma tarihini getir. ID'yi en sona koy.
                # Sonra kesme (cut) komutu ile sadece ID'yi al. Bu awk'tan daha güvenlidir.
                IMAGES_TO_DELETE=\$(
                    docker images --filter "reference=${REPO_NAME}:*" -a \
                    --format "{{.CreatedAt}}\t{{.ID}}" | sort -r | tail -n +4 | awk '{print \$NF}'
                )

                if [ -z "\$IMAGES_TO_DELETE" ]; then
                    echo "Silinecek eski proje imajı bulunamadı."
                else
                    echo "Silinecek imaj ID'leri: \$IMAGES_TO_DELETE"
                    # --no-run-if-empty yerine -r kullanıyoruz.
                    echo "\$IMAGES_TO_DELETE" | xargs -r docker rmi -f
                    echo "Eski proje imajları başarıyla temizlendi."
                fi
                
                # Kullanılmayan konteyner, network ve volume'leri temizle (Güvenli Prune)
                echo "Kullanılmayan genel Docker nesneleri temizleniyor..."
                docker container prune -f
                docker network prune -f 
                docker volume prune -f
            """
            }
        }
    }


}