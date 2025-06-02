pipeline {

    agent none // Tidak ada agent global, tentukan per stage



    environment {

        DOCKERHUB_CREDENTIALS_ID = 'dockerhub-credentials' // ID kredensial DockerHub di Jenkins



        // === PASTIKAN ANDA MENGGANTI PLACEHOLDER INI DENGAN INFO ANDA ===

        DOCKERHUB_USERNAME       = "muflihf" // GANTI dengan username Docker Hub Anda

        DOCKER_IMAGE_REPO_NAME   = "carvilla-app-wsl-final" // GANTI dengan nama repo image yang Anda inginkan di Docker Hub

        

        // Disesuaikan berdasarkan metadata.name di kubernetes/deployment.yaml Anda

        APP_DEPLOYMENT_NAME      = 'carvilla-web' 

        

        // Namespace tempat aplikasi akan di-deploy. File YAML aplikasi Anda menggunakan 'default'.

        APP_NAMESPACE            = 'default' 

        // =================================================================



        KUBERNETES_DEPLOYMENT_FILE = 'kubernetes/deployment.yaml' // Path sudah disesuaikan

        KUBERNETES_SERVICE_FILE    = 'kubernetes/service.yaml'  // Path sudah disesuaikan (asumsi)

    }



    stages {

        stage('Checkout SCM') {

            agent any // Bisa agent Jenkins default/master untuk checkout awal

            steps {

                echo "Checking out SCM (Source Code Management)..."

                checkout scm // Mengambil kode dari Git repository yang dikonfigurasi di Jenkins job

                echo "Checkout complete."

            }

        }



        stage('Run Tests (Placeholder)') {

            agent {

                kubernetes {

                    cloud 'k8s'                 // Nama Kubernetes Cloud Anda di Jenkins

                    inheritFrom 'jenkins-agent' // Merujuk ke NAMA Pod Template Anda

                }

            }

            steps {

                // 'jnlp' adalah nama container utama di pod agent Kubernetes

                container('jnlp') { 

                    echo 'Running tests... (No actual tests configured in this example)'

                    // Jika ada perintah tes, tambahkan di sini.

                }

            }

        }



        stage('Build and Push Docker Image') {

            agent {

                kubernetes {

                    cloud 'k8s'

                    inheritFrom 'jenkins-agent'

                }

            }

            steps {

                // Perintah Docker akan dijalankan di dalam container 'docker'

                container('docker') { 

                    script {

                        def imageTag = env.BUILD_NUMBER

                        def fullImageName = "${env.DOCKERHUB_USERNAME}/${env.DOCKER_IMAGE_REPO_NAME}:${imageTag}"



                        echo "Building Docker image: ${fullImageName}"

                        sh "docker build -t ${fullImageName} ."

                        echo "Docker image built."



                        echo "Logging in to Docker Hub..."

                        withCredentials([usernamePassword(credentialsId: env.DOCKERHUB_CREDENTIALS_ID, passwordVariable: 'DOCKERHUB_PASSWORD', usernameVariable: 'DOCKERHUB_USERNAME_FROM_CREDS')]) {

                            sh "echo \$DOCKERHUB_PASSWORD | docker login -u \$DOCKERHUB_USERNAME_FROM_CREDS --password-stdin"

                        }

                        echo "Logged in to Docker Hub."



                        echo "Pushing Docker image: ${fullImageName}"

                        sh "docker push ${fullImageName}"

                        echo "Docker image pushed."



                        echo "Updating Kubernetes deployment manifest (${env.KUBERNETES_DEPLOYMENT_FILE}) with image: ${fullImageName}"

                        sh "sed -i 's|^ *image:.*|  image: ${fullImageName}|g' ${env.KUBERNETES_DEPLOYMENT_FILE}"

                        echo "Kubernetes deployment manifest updated."



                        echo "Stashing Kubernetes manifests..."

                        stash includes: "${env.KUBERNETES_DEPLOYMENT_FILE}, ${env.KUBERNETES_SERVICE_FILE}", name: 'kubeManifestsForDeploy'

                        echo "Kubernetes manifests stashed."

                    }

                }

            }

        }



        stage('Deploy to Kubernetes') {

            agent {

                kubernetes {

                    cloud 'k8s'

                    inheritFrom 'jenkins-agent'

                }

            }

            steps {

                // Perintah kubectl akan dijalankan di dalam container 'kubectl'

                container('kubectl') { 

                    script {

                        echo "Unstashing Kubernetes manifests..."

                        unstash 'kubeManifestsForDeploy'

                        echo "Kubernetes manifests unstashed."



                        echo "Verifying kubectl context..."

                        sh 'kubectl version --client'



                        echo "Applying Kubernetes deployment (${env.KUBERNETES_DEPLOYMENT_FILE})..."

                        sh "kubectl apply -f ${env.KUBERNETES_DEPLOYMENT_FILE}"

                        echo "Kubernetes deployment applied."



                        echo "Applying Kubernetes service (${env.KUBERNETES_SERVICE_FILE})..."

                        sh "kubectl apply -f ${env.KUBERNETES_SERVICE_FILE}"

                        echo "Kubernetes service applied."



                        echo "Deployment to Kubernetes finished."

                        echo "Waiting for rollout of deployment '${env.APP_DEPLOYMENT_NAME}' in namespace '${env.APP_NAMESPACE}'..."

                        sh "kubectl rollout status deployment/${env.APP_DEPLOYMENT_NAME} -n ${env.APP_NAMESPACE} --timeout=5m" // Timeout dinaikkan sedikit

                    }

                }

            }

        }

    }



    post {

        always {

            echo 'Pipeline finished.'

            // Membersihkan workspace di agent yang menjalankan post-actions.

            // Menggunakan agent 'master' (built-in Jenkins node) untuk cleanup.

            node('master') { // Pastikan node Jenkins master/controller Anda memiliki label 'master'

               cleanWs()

            }

        }

        success {

            echo 'Pipeline Succeeded! ✅ Hooray!'

        }

        failure {

            echo 'Pipeline Failed. ❌ Oh no!'

        }

    }

}
