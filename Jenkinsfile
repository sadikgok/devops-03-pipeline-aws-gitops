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

                // 🧹 Remote Docker Hub Cleanup Stage
        stage("DockerHub Remote Cleanup") {
            steps {
                script {
                    echo "🌐 Docker Hub'daki eski imajlar kontrol ediliyor (latest korunacak)..."

                    // Jenkins credentials içindeki Docker Hub Access Token'ı alıyoruz
                    withCredentials([string(credentialsId: 'dockerhub-token', variable: 'DOCKER_HUB_TOKEN')]) {

                        def REPO = "sadikgok/devops-03-pipeline-aws-gitops"
                        def DAYS = 10  // 10 günden eski tag’ler silinecek
                        def API_URL = "https://hub.docker.com/v2/repositories/${REPO}/tags/?page_size=100"

                        sh """
                            echo "🔍 Docker Hub API çağrısı yapılıyor..."
                            curl -s -H "Authorization: Bearer ${DOCKER_HUB_TOKEN}" ${API_URL} > tags.json || true

                            if [ ! -s tags.json ]; then
                                echo "⚠️  Tag listesi alınamadı veya boş döndü."
                                exit 0
                            fi

                            echo "🧮 Eski tag'ler filtreleniyor..."
                            cat tags.json | jq -r '.results[] | [.name, .last_updated] | @tsv' | while IFS=$'\\t' read -r tag date; do
                                if [ "\$tag" = "latest" ]; then
                                    echo "⏩ 'latest' tag atlanıyor."
                                    continue
                                fi

                                # ISO tarih formatını epoch'a çevir
                                tag_date=\$(date -d "\$date" +%s 2>/dev/null || true)
                                now_date=\$(date +%s)
                                days_old=\$(( (now_date - tag_date) / 86400 ))

                                if [ \$days_old -gt ${DAYS} ]; then
                                    echo "🗑️  Siliniyor: \$tag (\$days_old gün önce oluşturulmuş)"
                                    curl -s -X DELETE -H "Authorization: Bearer ${DOCKER_HUB_TOKEN}" "https://hub.docker.com/v2/repositories/${REPO}/tags/\$tag/" || true
                                else
                                    echo "✅ \$tag tag'i yeni (\$days_old gün), korunuyor."
                                fi
                            done

                            echo "✨ Docker Hub temizliği tamamlandı."
                        """
                    }
                }
            }
        }

    }

    post {
        always {
            echo "📢 Trivy raporu yayınlanıyor..."
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
}
