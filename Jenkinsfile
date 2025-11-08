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
        JENKINS_API_TOKEN = credentials('JENKINS_API_TOKEN')

        // Trivy rapor dosyalarının dinamik isimleri
        TRIVY_JSON_REPORT = "trivy-report-${IMAGE_TAG}.json"
        TRIVY_HTML_REPORT = "trivy-report-${IMAGE_TAG}.html"
    }

    stages {

        stage('SCM GitHub') {
            steps {
                checkout scmGit(
                    branches: [[name: '*/main']],
                    extensions: [],
                    userRemoteConfigs: [[url: 'https://github.com/sadikgok/devops-03-pipeline-aws-gitops']]
                )
            }
        }
/*
        stage('Test Maven') {
            steps {
                script {
                    if (isUnix()) {
                        sh 'mvn test'
                    } else {
                        bat 'mvn test'
                    }
                }
            }
        }
*/
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
/*
        stage("SonarQube Analysis") {
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'SonarTokenForJenkins') {
                        if (isUnix()) {
                            sh "mvn sonar:sonar"
                        } else {
                            bat 'mvn sonar:sonar'
                        }
                    }
                }
            }
        }

        stage("Quality Gate") {
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
                    echo "Trivy Dosya Sistemi Taraması Başlatılıyor..."
                    def trivy_fs_command = "docker run --rm -v ${WORKSPACE}:/scan aquasec/trivy fs /scan --no-progress > trivyfs.txt"
                    if (isUnix()) {
                        sh trivy_fs_command
                    } else {
                        bat trivy_fs_command.replace('/scan', 'C:/scan')
                    }
                    echo "Trivy Dosya Sistemi Taraması Tamamlandı."
                }
            }
        }
 */
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
/*
        // 3. AŞAMA: Trivy İmaj Taraması ve Rapor
        stage("Trivy Image Scan - JSON + HTML") {
            steps {
                script {
                    def imageToScan = "${IMAGE_NAME}:${IMAGE_TAG}"
                    echo "Taranacak imaj: ${imageToScan}"

                    // İzin sorunlarını engelle
                    sh "chmod 777 ${WORKSPACE} || true"

                    // JSON rapor oluşturma
                    sh """
                        docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        -v ${WORKSPACE}:/report \
                        aquasec/trivy:0.67.2 image \
                        --format json \
                        --output /report/${TRIVY_JSON_REPORT} \
                        ${imageToScan}
                    """

                    // HTML rapor oluşturma (Trivy 0.67.2'de 'convert' komutu ile)
                    sh """
                        docker run --rm \
                        -v ${WORKSPACE}:/report \
                        aquasec/trivy:0.67.2 convert \
                        --format template \
                        --template "@/contrib/html.tpl" \
                        --output /report/${TRIVY_HTML_REPORT} \
                        /report/${TRIVY_JSON_REPORT}
                    """

                    echo "✅ Trivy JSON ve HTML raporları oluşturuldu."
                    sh "ls -lh ${WORKSPACE} | grep trivy-report || true"
                }
            }
        }
       */

        stage("Trigger CD Pipeline") {
            steps {
                script {
                    sh "curl -v -k --user sadikgok:${JENKINS_API_TOKEN} -X POST -H 'cache-control: no-cache' -H 'content-type: application/x-www-form-urlencoded' --data 'IMAGE_TAG=${IMAGE_TAG}' 'ec2-3-94-121-61.compute-1.amazonaws.com:8080/job/devops-03-pipeline-ArgoCD/buildWithParameters?token=GITOPS_TRIGGER_START'"
                }
            }
        }

        stages {
    // ... (Diğer stage'leriniz buraya gelir: Build, Test, Push, Trigger)

    stage('Cleanup Old Docker Tags') {
        environment {
            // Kaç tane son sürümü tutmak istediğiniz
            KEEP_COUNT = 3 
        }
        steps {
            script {
                // Jenkins Credentials'ları (Kullanıcı adı ve PAT) ile kimlik doğrulaması yapılır
                withCredentials([usernamePassword(
                    credentialsId: env.DOCKER_ID_LOGIN, 
                    usernameVariable: 'HUB_USER', 
                    passwordVariable: 'HUB_PAT'
                )]) {
                    sh """
                        set -euo pipefail
                        
                        # IMAGE_NAME değişkeni env bloğunda zaten tanımlı: sadikgok/devops-03-pipeline-aws-gitops
                        REPO_NAME="${env.IMAGE_NAME}"
                        
                        echo "1. Docker Hub JWT tokenı alınıyor..."
                        # Docker Hub JWT tokenını al
                        HUB_TOKEN=\$(curl -s -H "Content-Type: application/json" -X POST \\
                            -d "{\\"username\\": \\"\$HUB_USER\\", \\"password\\": \\"\$HUB_PAT\\"}" \\
                            https://hub.docker.com/v2/users/login/ | jq -r .token)

                        if [ -z "\$HUB_TOKEN" ]; then
                            echo "Hata: Docker JWT tokenı alınamadı. Kimlik bilgilerini kontrol edin."
                            exit 1
                        fi
                        
                        echo "2. Depodaki tüm etiketler çekiliyor..."
                        ALL_TAGS=\$(curl -s -H "Authorization: JWT \${HUB_TOKEN}" \\
                            "https://hub.docker.com/v2/repositories/\$REPO_NAME/tags/?page_size=1000" | jq -r '.results[].name')

                        if [ -z "\$ALL_TAGS" ]; then
                            echo "Uyarı: Depoda (\${REPO_NAME}) etiket bulunamadı."
                            exit 0
                        fi

                        echo "3. Etiketler sıralanıyor ve en son ${env.KEEP_COUNT} tanesi hariç tutuluyor..."
                        
                        # Etiketleri Versiyon (sort -V) ve tersten (sort -rV) sırala.
                        # En yeni ${KEEP_COUNT} tanesini (tail -n +$) atla.
                        TAGS_TO_DELETE=\$(echo "\$ALL_TAGS" | sort -rV | tail -n +\$(( ${env.KEEP_COUNT} + 1 )))
                        
                        if [ -z "\$TAGS_TO_DELETE" ]; then
                            echo "Silinecek eski sürüm bulunamadı. (${env.KEEP_COUNT} sürüm korunuyor)"
                            exit 0
                        fi

                        echo "4. Silinecek Etiketler: \n\${TAGS_TO_DELETE}"

                        echo "5. Etiketler siliniyor..."
                        
                        echo "\$TAGS_TO_DELETE" | while read TAG; do
                            echo "  -> Siliniyor: \${TAG}"
                            curl -s -X DELETE \\
                                -H "Authorization: JWT \${HUB_TOKEN}" \\
                                "https://hub.docker.com/v2/repositories/\$REPO_NAME/tags/\${TAG}/"
                        done
                        
                        echo "Temizleme işlemi tamamlandı."
                    """
                }
            }
        }
    }
}

    }
/*
    post {
        always {
            // echo " Trivy raporu yayınlanıyor..."
            // publishHTML(
            //     target: [
            //         allowMissing         : false,
            //         alwaysLinkToLastBuild: true,
            //         keepAll              : true,
            //         reportDir            : "${WORKSPACE}",
            //         reportFiles          : "${TRIVY_HTML_REPORT}",
            //         reportName           : "Trivy Security Report - ${IMAGE_TAG}"
            //     ]
            // )
        }

        success {
            echo "✅ Pipeline başarılı. Docker imajları temizleniyor..."
            script {
                def REPO_NAME = "${IMAGE_NAME}"
                sh """
                    echo "🧹 Eski imajlar için temizlik başlatılıyor (Son 3 imaj korunacak)..."
                    IMAGES_TO_DELETE=\$(
                        docker images --filter "reference=${REPO_NAME}:*" -a \
                        --format "{{.CreatedAt}}\\t{{.ID}}" | sort -r | tail -n +4 | awk '{print \$NF}'
                    )

                    if [ -z "\$IMAGES_TO_DELETE" ]; then
                        echo "Silinecek eski proje imajı bulunamadı."
                    else
                        echo "Silinecek imaj ID'leri: \$IMAGES_TO_DELETE"
                        echo "\$IMAGES_TO_DELETE" | xargs -r docker rmi -f
                    fi

                    echo "🧽 Gerçekten tüm <none> imajlar temizleniyor..."
                    docker images -f "dangling=true" -q | xargs -r docker rmi -f || true

                    echo "🧽 Label'sız veya bozuk <none> imajlar da temizleniyor..."
                    docker images | grep '<none>' | awk '{print \$3}' | xargs -r docker rmi -f || true

                    echo "🧹 Kullanılmayan Docker nesneleri temizleniyor..."
                    docker container prune -f
                    docker network prune -f 
                    docker volume prune -f

                    echo "✨ Docker temizlik tamamlandı."
                """
            }
        }
    }
    */
}
