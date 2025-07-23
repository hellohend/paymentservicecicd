// Jenkinsfile
// -------------------------------
// Jalankan pada Jenkins agent apa pun yang memiliki 'oc' CLI
// Pastikan Jenkins memiliki akses ke cluster (service‑account di dalam cluster
//   atau oc login --token=… untuk agent di luar cluster).

pipeline {
    agent any

    environment {
        // Ganti keduanya sesuai cluster Anda
        OPENSHIFT_PROJECT = 'payment-service1'     // namespace / project
        APP_NAME          = 'payment-service'       // nama app, buildconfig, imagestream, dsb.
    }

    stages {

        // -----------------------------------------------------------------
        stage('Checkout') {
            steps {
                echo 'Checking out source code…'
                checkout scm                        // pull repo yang berisi Dockerfile, src, Jenkinsfile
            }
        }

        // -----------------------------------------------------------------
        stage('Build & Test') {
            steps {
                echo 'Building application & running tests…'
                script {
                    sh 'chmod +x ./gradlew'
                    sh './gradlew clean build'      // menghasilkan JAR di build/libs/…
                }
            }
        }

        // -----------------------------------------------------------------
        stage('Ensure BuildConfig') {
            steps {
                script {
                    echo 'Checking BuildConfig…'
                    def bcExists = sh(
                        script: "oc get bc ${APP_NAME} -n ${OPENSHIFT_PROJECT} --ignore-not-found",
                        returnStatus: true
                    ) == 0

                    if (!bcExists) {
                        echo "BuildConfig ${APP_NAME} not found – creating it."
                        sh """
                           oc new-build --name=${APP_NAME}          \
                                        --binary=true               \
                                        --strategy=docker           \
                                        --to=${APP_NAME}:latest     \
                                        -n ${OPENSHIFT_PROJECT}
                        """
                    } else {
                        echo "BuildConfig ${APP_NAME} already exists."
                    }
                }
            }
        }

        // -----------------------------------------------------------------
        stage('Build OpenShift Image') {
            steps {
                script {
                    echo "Building container image in OpenShift…"
                    // upload entire repo (Dockerfile + artefak JAR) sebagai binary build:
                    sh "oc start-build ${APP_NAME} --from-dir=. --follow -n ${OPENSHIFT_PROJECT}"
                }
            }
        }

        // -----------------------------------------------------------------
        stage('Deploy to OpenShift') {
            steps {
                script {
                    echo "Ensuring Deployment exists…"
                    def deployExists = sh(
                        script: "oc get deployment ${APP_NAME} -n ${OPENSHIFT_PROJECT} --ignore-not-found",
                        returnStatus: true
                    ) == 0

                    if (!deployExists) {
                        // Buat Deployment + Service baru dari ImageStream hasil build
                        sh """
                           oc new-app --image-stream=${OPENSHIFT_PROJECT}/${APP_NAME}:latest \
                                      --name=${APP_NAME}                                   \
                                      -n ${OPENSHIFT_PROJECT}
                        """
                    } else {
                        // Trigger rollout untuk image terbaru
                        sh "oc rollout latest deployment/${APP_NAME} -n ${OPENSHIFT_PROJECT}"
                    }

                    echo "Waiting for rollout to finish…"
                    sh "oc rollout status deployment/${APP_NAME} -n ${OPENSHIFT_PROJECT}"
                }
            }
        }

        // -----------------------------------------------------------------
        stage('Expose Service') {
            steps {
                script {
                    echo "Exposing service as Route…"
                    def routeExists = sh(
                        script: "oc get route ${APP_NAME} -n ${OPENSHIFT_PROJECT} --ignore-not-found",
                        returnStatus: true
                    ) == 0

                    if (!routeExists) {
                        sh "oc expose service/${APP_NAME} --name=${APP_NAME} -n ${OPENSHIFT_PROJECT}"
                        echo "Route created."
                    } else {
                        echo "Route already exists."
                    }

                    // Ambil host dan tampilkan
                    def host = sh(
                        script: "oc get route ${APP_NAME} -n ${OPENSHIFT_PROJECT} -o jsonpath='{.spec.host}'",
                        returnStdout: true
                    ).trim()
                    echo "✅ Application is available at: http://${host}"
                }
            }
        }
    }

    // ---------------------------------------------------------------------
    post {
        always {
            echo 'Pipeline finished.'
            // Batalkan build state NEW/ PENDING agar tak numpuk (abaikan error jika BC belum ada)
            sh "oc cancel-build bc/${APP_NAME} --state=new -n ${OPENSHIFT_PROJECT} || true"
        }
    }
}
