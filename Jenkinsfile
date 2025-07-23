
pipeline {
    agent any

    environment {
        // Ganti dengan nama project (namespace) di OpenShift Anda
        OPENSHIFT_PROJECT = 'your-openshift-project'
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
                    // Ini akan mengompilasi kode, menjalankan unit test, dan membuat file JAR
                    sh './gradlew clean build'
                }
            }
        }

        stage('Build OpenShift Image') {
            steps {
                script {
                    echo "Logging into OpenShift..."
                    // Pastikan Jenkins memiliki kredensial yang tepat untuk login
                    // Jenkins akan menggunakan service account token yang ter-mount secara default jika berjalan di dalam cluster
                    // Jika tidak, konfigurasikan `oc login` dengan token atau kredensial yang sesuai
                    
                    echo "Starting Docker build on OpenShift..."
                    // Menggunakan `oc start-build` dengan strategi Docker
                    // `--from-dir=.` akan meng-upload konteks direktori saat ini (termasuk Dockerfile dan file JAR yang sudah di-build) ke OpenShift
                    // `--follow` akan menunggu build selesai dan menampilkan log-nya
                    sh "oc start-build ${APP_NAME} --from-dir=. --follow -n ${OPENSHIFT_PROJECT}"
                }
            }
        }

        stage('Deploy to OpenShift') {
            steps {
                script {
                    echo "Deploying application..."
                    // `oc rollout latest` akan memicu deployment baru dengan image yang baru saja di-build
                    // Ini akan menyebabkan OpenShift membuat Pod baru dengan versi aplikasi yang terbaru
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
                    // Mengecek apakah route sudah ada
                    def routeExists = sh(script: "oc get route ${APP_NAME} -n ${OPENSHIFT_PROJECT} --ignore-not-found", returnStatus: true) == 0
                    if (!routeExists) {
                        // Jika belum ada, buat route baru agar aplikasi bisa diakses dari luar cluster
                        sh "oc expose service/${APP_NAME} --name=${APP_NAME} -n ${OPENSHIFT_PROJECT}"
                        echo "Route created. Access your application at: $(oc get route ${APP_NAME} -n ${OPENSHIFT_PROJECT} -o jsonpath='{.spec.host}')"
                    } else {
                        echo "Route already exists. Access your application at: $(oc get route ${APP_NAME} -n ${OPENSHIFT_PROJECT} -o jsonpath='{.spec.host}')"
                    }
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
