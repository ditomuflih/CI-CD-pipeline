pipeline {

    agent none // Tidak ada agent global, tentukan per stage



    environment {

        DOCKERHUB_CREDENTIALS_ID = 'dockerhub-credentials' // ID kredensial DockerHub di Jenkins



        // === PASTIKAN ANDA MENGGANTI PLACEHOLDER INI DENGAN INFO ANDA ===

        DOCKERHUB_USERNAME       = "muflihf" // GANTI dengan username Docker Hub Anda

        DOCKER_IMAGE_REPO_NAME   = "carvilla-app-wsl-final" // GANTI dengan nama repo image yang Anda inginkan di Docker Hub

        

        APP_DEPLOYMENT_NAME      = 'carvilla-web' // DISESUAIKAN berdasarkan file YAML aplikasi Anda

        APP_NAMESPACE            = 'default' 

        // =================================================================



        KUBERNETES_DEPLOYMENT_FILE = 'kubernetes/deployment.yaml'

        KUBERNETES_SERVICE_FILE    = 'kubernetes/service.yaml'

    }



    stages {

        stage('Checkout SCM') {

            agent any 

            steps {

                echo "Checking out SCM (Source Code Management)..."

                checkout scm 

                echo "Checkout complete."

            }

        }



        stage('Run Tests (Placeholder)') {

            agent {

                kubernetes {

                    cloud 'k8s'                

                    inheritFrom 'jenkins-agent' 

                }

            }

            steps {

                container('jnlp') { 

                    echo 'Running tests... (No actual tests configured in this example)'

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



        stage('Deploy to Kubernetes (Diagnostic Mode)') { // Nama stage diubah untuk menandakan mode diagnostik

            agent {

                kubernetes {

                    cloud 'k8s'

                    inheritFrom 'jenkins-agent'

                }

            }

            steps {

                // Mencoba menjalankan perintah di container default (jnlp) dari Pod Template 'jenkins-agent'

                container('jnlp') { 

                    script {

                        echo "DIAGNOSTIC: Unstashing Kubernetes manifests into JNLP container..."

                        unstash 'kubeManifestsForDeploy'

                        echo "DIAGNOSTIC: Kubernetes manifests unstashed into JNLP container."



                        echo "DIAGNOSTIC: Testing basic command execution in JNLP container..."

                        sh 'echo "Hello from JNLP container!"'

                        sh 'pwd'

                        sh 'ls -la'

                        sh 'ls -la kubernetes/' // Untuk melihat apakah hasil unstash ada di sini



                        echo "DIAGNOSTIC: Attempting to install kubectl in JNLP container (assuming Alpine base)..."

                        // Perintah instalasi kubectl untuk Alpine (image jenkins/inbound-agent biasanya Alpine)

                        // Jika gagal di sini, berarti image jnlp tidak bisa/tidak diizinkan melakukan ini,

                        // atau tool (seperti apk, curl) tidak ada.

                        sh '''

                        echo "--- Starting kubectl install ---"

                        apk update && apk add --no-cache curl gettext && \\

                        echo "curl and gettext installed (or already present)." && \\

                        curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && \\

                        echo "kubectl downloaded." && \\

                        chmod +x kubectl && \\

                        mv kubectl /usr/local/bin/kubectl && \\

                        echo "kubectl moved to /usr/local/bin." && \\

                        kubectl version --client

                        echo "--- Kubectl install attempt finished ---"

                        '''



                        // Perintah apply asli dikomentari untuk sekarang

                        // echo "Applying Kubernetes deployment (${env.KUBERNETES_DEPLOYMENT_FILE})..."

                        // sh "kubectl apply -f ${env.KUBERNETES_DEPLOYMENT_FILE}"

                        // echo "Kubernetes deployment applied."



                        // echo "Applying Kubernetes service (${env.KUBERNETES_SERVICE_FILE})..."

                        // sh "kubectl apply -f ${env.KUBERNETES_SERVICE_FILE}"

                        // echo "Kubernetes service applied."



                        // echo "Deployment to Kubernetes finished."

                        // echo "Waiting for rollout of deployment '${env.APP_DEPLOYMENT_NAME}' in namespace '${env.APP_NAMESPACE}'..."

                        // sh "kubectl rollout status deployment/${env.APP_DEPLOYMENT_NAME} -n ${env.APP_NAMESPACE} --timeout=3m"

                        echo "DIAGNOSTIC: Deploy stage finished. Actual kubectl apply commands were commented out."

                    }

                }

            }

        }

    }



    post {

        always {

            echo 'Pipeline finished.'

            node('master') { 

               cleanWs()

            }

        }

        success {

            echo 'Pipeline Succeeded! ✅ (Diagnostic Mode)'

        }

        failure {

            echo 'Pipeline Failed. ❌ (Diagnostic Mode)'

        }

    }

}
