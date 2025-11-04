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
        stage("Trivy File System Scan"){
            steps{
                script {
                    if (isUnix()) {
                        withEnv(["TRIVY_CACHE_DIR=${WORKSPACE}/.trivy-cache"]) {
                            sh 'trivy fs . > trivyfs.txt'
                        }
                    } else {
                        bat 'set TRIVY_CACHE_DIR=%cd%\\.trivy-cache && trivy fs . > trivyfs.txt'
                    }
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
                    
                    withEnv([
                        "TRIVY_CACHE_DIR=${WORKSPACE}/.trivy-cache",
                        "TRIVY_TEMP_DIR=${WORKSPACE}/.trivy-temp"  
                    ]) {
                        echo "Taranacak İmaj: ${imageToScan}"
                        
                        // Önemli: YÜKSEK (HIGH) ve KRİTİK (CRITICAL) açık bulunursa pipeline durur (exit code 1)
                        // Çıktı JSON formatında raporlama için dosyaya kaydedilir.
                        sh "trivy image --exit-code 1 --severity CRITICAL,HIGH --format json --output ${TRIVY_JSON_REPORT} ${imageToScan}"
                    }
                }
            }
        }
        
        // 4. AŞAMA: HTML Raporu Oluşturma
        stage("Generate HTML Report"){
            steps {
                // Not: Bu komutun çalışması için Trivy'nin 'html.tpl' şablonuna erişimi olmalıdır.
                // Eğer hata alırsanız, harici bir dönüştürücü betik kullanmanız gerekebilir.
                sh "trivy convert --format template --template '@contrib/html.tpl' ${TRIVY_JSON_REPORT} --output ${TRIVY_HTML_REPORT}"
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
            echo "Trivy raporu yayımlanıyor..."
            
            // HTML Publisher Plugin kullanımı
            publishHTML(
                target: [
                    allowMissing         : false, 
                    alwaysLinkToLastBuild: true,  
                    keepAll              : true,  
                    reportDir            : "${WORKSPACE}",
                    reportFiles          : "${TRIVY_HTML_REPORT}",
                    reportName           : "Trivy Güvenlik Raporu - ${IMAGE_TAG}"
                ]
            )
        }
    }

}