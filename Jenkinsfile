pipeline {
    agent any

    environment {
        // Ganti dengan nama project (namespace) di OpenShift Anda
        OPENSHIFT_PROJECT = 'payment-service1'
        // Nama aplikasi yang akan digunakan untuk semua resource (Deployment, Service, Route, etc)
        APP_NAME = 'payment-service'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }

        stage('Build & Test') {
            steps {
                echo 'Building the application and running tests...'
                script {
                    // Memberikan izin eksekusi pada gradlew
                    sh 'chmod +x ./gradlew'
                    // Menjalankan clean build untuk memastikan tidak ada artefak lama
                    sh './gradlew clean build'
                }
            }
        }

        stage('Build OpenShift Image') {
            steps {
                script {
                    echo "Logging into OpenShift..."
                    // Jika diperlukan, tambahkan oc login di sini

                    echo "Starting Docker build on OpenShift..."
                    sh "oc start-build ${APP_NAME} --from-dir=. --follow -n ${OPENSHIFT_PROJECT}"
                }
            }
        }

        stage('Deploy to OpenShift') {
            steps {
                script {
                    echo "Deploying application..."
                    sh "oc rollout latest deployment/${APP_NAME} -n ${OPENSHIFT_PROJECT}"
                    echo "Waiting for deployment to complete..."
                    sh "oc rollout status deployment/${APP_NAME} -n ${OPENSHIFT_PROJECT}"
                }
            }
        }

        stage('Expose Service') {
            steps {
                script {
                    echo "Exposing service as a route..."
                    // Cek apakah route sudah ada
                    def routeExists = sh(
                        script: "oc get route ${APP_NAME} -n ${OPENSHIFT_PROJECT} --ignore-not-found",
                        returnStatus: true
                    ) == 0

                    if (!routeExists) {
                        // Buat route baru
                        sh "oc expose service/${APP_NAME} --name=${APP_NAME} -n ${OPENSHIFT_PROJECT}"
                        echo "Route created."
                    } else {
                        echo "Route already exists."
                    }

                    // Ambil host dari route lalu tampilkan
                    def host = sh(
                        script: "oc get route ${APP_NAME} -n ${OPENSHIFT_PROJECT} -o jsonpath='{.spec.host}'",
                        returnStdout: true
                    ).trim()
                    echo "Access your application at: ${host}"
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished.'
            // Membersihkan build yang tidak perlu di OpenShift
            sh "oc cancel-build bc/${APP_NAME} --state=new -n ${OPENSHIFT_PROJECT}"
        }
    }
}
