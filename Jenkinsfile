// Jenkinsfile
// -----------------------------------------------------------------------------
// • Build Spring/Gradle project ➜ menghasilkan JAR
// • Trigger/kelola BuildConfig & ImageStream ➜ tag 'latest' + SEMANTIC_VERSION
// • Apply manifest OpenShift ➜ rahasia, service, deployment, dsb.
// • Set image ke deployment & rollout
//
// Asumsi: Jenkins agent punya 'oc' CLI & kredensial login ke cluster.
//

pipeline {
    agent any

    /*--- PARAMETERS ---------------------------------------------------------*/
    // Anda bisa mendorong versi semver via parameter; default memakai BUILD_NUMBER
    parameters {
        string(
            name: 'SEMANTIC_VERSION',
            defaultValue: '',
            description: 'Tag semver untuk image (contoh 1.0.3). Kosongkan untuk pakai BUILD_NUMBER.'
        )
    }

    /*--- ENVIRONMENT --------------------------------------------------------*/
    environment {
        // Ganti sesuai cluster
        NAMESPACE = 'payment-service1'            // project / namespace OpenShift
        APP_NAME  = 'payment-service'             // nama BuildConfig & Deployment
        // alamat registry internal OpenShift (ubah bila beda)
        REGISTRY  = "image-registry.openshift-image-registry.svc:5000"
        // Hitung SEMVER: pakai param jika ada, else fallback ke BUILD_NUMBER
        SEMANTIC_VERSION = "${ (params.SEMANTIC_VERSION ?: env.BUILD_NUMBER) }"
    }

    /*--- STAGES -------------------------------------------------------------*/
    stages {
        // ---------------------------------------------------------------------
        stage('Checkout') {
            steps {
                echo '📥  Checking out source code…'
                checkout scm
            }
        }

        // ---------------------------------------------------------------------
        stage('Build & Test') {
            steps {
                echo '🔨  Building application & running tests…'
                script {
                    sh 'chmod +x ./gradlew'
                    sh './gradlew clean build'    // JAR di build/libs/…
                }
            }
        }

        // ---------------------------------------------------------------------
        stage('Build with OpenShift BuildConfig') {
            steps {
                script {
                    echo "🚀  Triggering OpenShift build for ${APP_NAME}:${SEMANTIC_VERSION}"

                    sh """
                        # Berpindah project
                        oc project ${NAMESPACE}

                        ################################################################
                        # 1) Buat / update BuildConfig
                        ################################################################
                        oc apply -f - <<EOF
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
  labels:
    app: ${APP_NAME}
spec:
  source:
    type: Binary                          # kita upload konteks repo (Dockerfile + JAR)
  strategy:
    type: Docker
    dockerStrategy:
      dockerfilePath: Dockerfile
  output:
    to:
      kind: ImageStreamTag
      name: ${APP_NAME}:latest
  triggers:                               # manual trigger saja
  - type: Manual
  runPolicy: Serial
EOF

                        ################################################################
                        # 2) Buat / update ImageStream
                        ################################################################
                        oc apply -f - <<EOF
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
  labels:
    app: ${APP_NAME}
spec:
  lookupPolicy:
    local: false
EOF

                        ################################################################
                        # 3) Start build: upload seluruh repo sebagai binary input
                        ################################################################
                        echo "⏳  Starting OpenShift build…"
                        oc start-build ${APP_NAME} --from-dir=. --wait --follow | cat

                        ################################################################
                        # 4) Tag hasil build dengan semantic version
                        ################################################################
                        oc tag ${APP_NAME}:latest ${APP_NAME}:${SEMANTIC_VERSION}

                        echo "✅  Build completed -> tags: latest & ${SEMANTIC_VERSION}"
                    """
                }
            }
        }

        // ---------------------------------------------------------------------
        stage('Apply OpenShift Resources') {
            steps {
                script {
                    echo '📜  Applying resource manifests…'
                    sh """
                        oc project ${NAMESPACE}

                        # Terapkan YAML di folder openshift/ (contoh: secrets, service, deployment)
                        oc apply -f openshift/secrets.yaml
                        oc apply -f openshift/service.yaml
                        oc apply -f openshift/deployment.yaml

                        echo '✅  Resources applied.'
                    """
                }
            }
        }

        // ---------------------------------------------------------------------
        stage('Deploy Application') {
            steps {
                script {
                    echo "🚢  Deploying ${APP_NAME} image…"

                    sh """
                        oc project ${NAMESPACE}

                        # Ganti image container pada Deployment ke tag 'latest'
                        oc set image deployment/${APP_NAME} ${APP_NAME}=${REGISTRY}/${NAMESPACE}/${APP_NAME}:latest -n ${NAMESPACE}

                        # Restart (rollout) deployment supaya pod baru tarik image
                        oc rollout restart deployment/${APP_NAME} -n ${NAMESPACE}

                        # Tunggu rollout selesai (max 5 menit)
                        oc rollout status deployment/${APP_NAME} -n ${NAMESPACE} --timeout=300s | cat

                        # Tampilkan status pod & image
                        echo ''
                        oc get pods -l app=${APP_NAME} -n ${NAMESPACE}
                        echo ''
                        echo 'Current deployment image:'
                        oc get deployment ${APP_NAME} -o jsonpath='{.spec.template.spec.containers[0].image}' -n ${NAMESPACE}
                        echo ''
                        echo '✅  Deployed tags: latest & ${SEMANTIC_VERSION}'
                        echo '🔍  ImageStream tags available:'
                        oc get imagestream ${APP_NAME} -o jsonpath='{.status.tags[*].tag}' -n ${NAMESPACE} | tr ' ' '\\n'
                    """
                }
            }
        }
    }

    /*--- POST ----------------------------------------------------------------*/
    post {
        success {
            echo """
🎉  Pipeline SUCCESS
    • Image tags  : latest, ${SEMANTIC_VERSION}
    • Namespace   : ${NAMESPACE}
    • Deployment  : ${APP_NAME}

Untuk port‑forward lokal:
    oc port-forward svc/${APP_NAME} 8080:8080 -n ${NAMESPACE}
"""
        }
        failure {
            script {
                echo '❌  Pipeline FAILED – lihat log di atas.'
                // try dumping last build logs (abaikan error jika tidak ada build)
                sh """
                    echo '─── Recent Build Logs (if any) ───'
                    oc logs -l build=${APP_NAME} --tail=50 -n ${NAMESPACE} || true
                """
            }
        }
    }
}
