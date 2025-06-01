pipeline {

    agent none // Tidak ada agent global, tentukan per stage



    environment {

        DOCKERHUB_CREDENTIALS_ID = 'dockerhub-credentials' // ID kredensial DockerHub di Jenkins



        // === GANTI BAGIAN DI BAWAH INI DENGAN INFO ANDA ===

        DOCKERHUB_USERNAME       = "muflihf" // GANTI dengan username Docker Hub Anda

        DOCKER_IMAGE_REPO_NAME   = "carvilla-app-wsl-final" // GANTI dengan nama repo image yang Anda inginkan di Docker Hub

        // Pastikan nama deployment di k8s/deployment.yaml (metadata.name) adalah 'carvilla' atau sesuaikan di bawah

        APP_DEPLOYMENT_NAME      = 'carvilla'

        // Sesuaikan namespace jika aplikasi Anda di-deploy ke namespace selain 'default'

        // File YAML di repo asli tidak menentukan namespace, jadi akan masuk ke 'default'.

        APP_NAMESPACE            = 'default'

        // =================================================



        KUBERNETES_DEPLOYMENT_FILE = 'kubernetes/deployment.yaml' // Path relatif dari root repo

        KUBERNETES_SERVICE_FILE    = 'kubernetes/service.yaml'     // Path relatif dari root repo

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

                    label 'jenkins-agent' // Menggunakan agent Kubernetes dari Pod Template

                    // cloud 'kubernetes' // Opsional: Nama Kubernetes cloud Anda jika bukan 'kubernetes'

                }

            }

            steps {

                // 'jnlp' adalah nama default container utama di pod agent Kubernetes

                container('jnlp') {

                    echo 'Running tests... (No actual tests configured in this example)'

                    // Jika ada perintah tes, tambahkan di sini.

                    // Contoh: sh './mvnw test' atau sh 'npm test'

                }

            }

        }



        stage('Build and Push Docker Image') {

            agent {

                kubernetes {

                    label 'jenkins-agent' // Harus cocok dengan label di Pod Template Anda

                    // cloud 'kubernetes' // Opsional: Nama Kubernetes cloud Anda

                }

            }

            steps {

                // Perintah Docker akan dijalankan di dalam container 'docker'

                // yang telah didefinisikan di Pod Template 'jenkins-agent'

                // dan memiliki akses ke Docker socket.

                container('docker') {

                    script {

                        def imageTag = env.BUILD_NUMBER

                        def fullImageName = "${env.DOCKERHUB_USERNAME}/${env.DOCKER_IMAGE_REPO_NAME}:${imageTag}"



                        echo "Building Docker image: ${fullImageName}"

                        // Dockerfile harus ada di root workspace hasil checkout

                        sh "docker build -t ${fullImageName} ."

                        echo "Docker image built."



                        echo "Logging in to Docker Hub..."

                        // Menggunakan kredensial yang sudah disimpan di Jenkins

                        withCredentials([usernamePassword(credentialsId: env.DOCKERHUB_CREDENTIALS_ID, passwordVariable: 'DOCKERHUB_PASSWORD', usernameVariable: 'DOCKERHUB_USERNAME_FROM_CREDS')]) {

                            sh "echo \$DOCKERHUB_PASSWORD | docker login -u \$DOCKERHUB_USERNAME_FROM_CREDS --password-stdin"

                        }

                        echo "Logged in to Docker Hub."



                        echo "Pushing Docker image: ${fullImageName}"

                        sh "docker push ${fullImageName}"

                        echo "Docker image pushed."



                        echo "Updating Kubernetes deployment manifest (${env.KUBERNETES_DEPLOYMENT_FILE}) with image: ${fullImageName}"

                        // Menggunakan sed untuk mengganti placeholder image di file deployment.yaml

                        sh "sed -i 's|^ *image:.*|  image: ${fullImageName}|g' ${env.KUBERNETES_DEPLOYMENT_FILE}"

                        echo "Kubernetes deployment manifest updated."



                        echo "Stashing Kubernetes manifests..."

                        // Stash file YAML yang sudah diubah untuk digunakan di stage deploy

                        stash includes: "${env.KUBERNETES_DEPLOYMENT_FILE}, ${env.KUBERNETES_SERVICE_FILE}", name: 'kubeManifestsForDeploy'

                        echo "Kubernetes manifests stashed."

                    }

                }

            }

        }



        stage('Deploy to Kubernetes') {

            agent {

                kubernetes {

                    label 'jenkins-agent' // Gunakan agent yang sama (memiliki kubectl via service account)

                    // cloud 'kubernetes' // Opsional

                }

            }

            steps {

                // Perintah kubectl akan dijalankan di container 'jnlp' (default dari pod agent)

                // yang seharusnya memiliki konteks Service Account untuk berinteraksi dengan K8s API.

                // Jika container 'jnlp' Anda (dari image jenkins/inbound-agent) tidak punya kubectl,

                // Anda perlu menambahkannya ke image tersebut atau menambahkan container lain (misal, bitnami/kubectl)

                // ke Pod Template dan menjalankan perintah kubectl di sana.

                // Untuk Minikube, service account Jenkins ('jenkins-sa') sudah punya hak.

                container('jnlp') {

                    script {

                        echo "Unstashing Kubernetes manifests..."

                        unstash 'kubeManifestsForDeploy'

                        echo "Kubernetes manifests unstashed."



                        echo "Verifying kubectl context (running inside Kubernetes, should be automatic)..."

                        // sh 'kubectl version --client' // Untuk verifikasi jika kubectl ada di 'jnlp'



                        echo "Applying Kubernetes deployment (${env.KUBERNETES_DEPLOYMENT_FILE})..."

                        sh "kubectl apply -f ${env.KUBERNETES_DEPLOYMENT_FILE}"

                        echo "Kubernetes deployment applied."



                        echo "Applying Kubernetes service (${env.KUBERNETES_SERVICE_FILE})..."

                        sh "kubectl apply -f ${env.KUBERNETES_SERVICE_FILE}"

                        echo "Kubernetes service applied."



                        echo "Deployment to Kubernetes finished."

                        echo "Waiting for rollout of deployment '${env.APP_DEPLOYMENT_NAME}' in namespace '${env.APP_NAMESPACE}'..."

                        // Perintah ini akan menunggu rollout selesai atau timeout

                        sh "kubectl rollout status deployment/${env.APP_DEPLOYMENT_NAME} -n ${env.APP_NAMESPACE} --timeout=3m"

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

            node('master') {

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
