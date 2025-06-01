
pipeline {
    agent none // Tidak ada agent global, tentukan per stage

    environment {
        DOCKERHUB_CREDENTIALS_ID = 'dockerhub-credentials' // ID kredensial DockerHub di Jenkins
        // === GANTI BAGIAN DI BAWAH INI DENGAN INFO ANDA ===
        DOCKERHUB_USERNAME = "muflihf" // GANTI dengan username Docker Hub Anda
        DOCKER_IMAGE_REPO_NAME = "carvilla-app" // GANTI dengan nama repo image yang Anda inginkan di Docker Hub
        // =================================================
        KUBERNETES_DEPLOYMENT_FILE = 'k8s/deployment.yaml'
        KUBERNETES_SERVICE_FILE = 'k8s/service.yaml'
    }

    stages {
        stage('Checkout') {
            agent any
            steps {
                echo "Checking out SCM..."
                checkout scm
                echo "Checkout complete."
            }
        }

        stage('Run Tests') {
            agent any
            steps {
                echo 'Running tests... (placeholder)'
                // Jika ada perintah tes, tambahkan di sini.
            }
        }

        stage('Build and Push Docker Image') {
            agent {
                docker {
                    image 'docker:20.10.24' // Atau versi stabil Docker CLI lainnya
                }
            }
            steps {
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
                    stash includes: "${env.KUBERNETES_DEPLOYMENT_FILE}, ${env.KUBERNETES_SERVICE_FILE}", name: 'kubeManifests'
                    echo "Kubernetes manifests stashed."
                }
            }
        }

        stage('Deploy to Kubernetes') {
            agent {
                // Menggunakan agent master Jenkins, dengan asumsi kubectl ada & terkonfigurasi
                // (karena Jenkins berjalan di dalam cluster, ini biasanya sudah benar)
                label 'master'
            }
            steps {
                script {
                    echo "Unstashing Kubernetes manifests..."
                    unstash 'kubeManifests'
                    echo "Kubernetes manifests unstashed."

                    echo "Verifying kubectl context..."
                    sh 'kubectl config current-context'

                    echo "Applying Kubernetes deployment (${env.KUBERNETES_DEPLOYMENT_FILE})..."
                    sh "kubectl apply -f ${env.KUBERNETES_DEPLOYMENT_FILE}"
                    echo "Kubernetes deployment applied."

                    echo "Applying Kubernetes service (${env.KUBERNETES_SERVICE_FILE})..."
                    sh "kubectl apply -f ${env.KUBERNETES_SERVICE_FILE}"
                    echo "Kubernetes service applied."

                    echo "Deployment to Kubernetes finished."
                    echo "Waiting for deployment rollout to complete..."
                    // Ambil nama deployment dari file YAML (asumsi hanya ada satu 'name:' di bawah 'metadata:' utama deployment)
                    // dan namespace jika ada
                    def deploymentName = sh(script: "yq e '.metadata.name' ${env.KUBERNETES_DEPLOYMENT_FILE}", returnStdout: true).trim()
                    def namespaceInfo = sh(script: "yq e '.metadata.namespace // \"default\"' ${env.KUBERNETES_DEPLOYMENT_FILE}", returnStdout: true).trim()
                    def namespaceArg = (namespaceInfo != "default" && namespaceInfo != "null") ? "-n ${namespaceInfo}" : ""

                    if (deploymentName && deploymentName != "null") {
                        echo "Waiting for rollout of deployment '${deploymentName}' in namespace '${namespaceInfo}'..."
                        sh "kubectl rollout status deployment/${deploymentName} ${namespaceArg} --timeout=3m"
                    } else {
                        echo "Could not determine deployment name from ${env.KUBERNETES_DEPLOYMENT_FILE} to check rollout status."
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished.'
            cleanWs() // Membersihkan workspace setelah selesai
        }
        success {
            echo 'Pipeline succeeded! Hooray! 🎉'
        }
        failure {
            echo 'Pipeline failed. 😥'
        }
    }
}
