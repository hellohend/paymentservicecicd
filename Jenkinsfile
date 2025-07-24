// Jenkinsfile -----------------------------------------------------------------
pipeline {
    agent any

    /* PARAMETERS ----------------------------------------------------------- */
    parameters {
        string(name: 'SEMANTIC_VERSION',
               defaultValue: '',
               description: 'Tag semver (mis. 1.0.3). Kosong = BUILD_NUMBER')
    }

    /* ENV ------------------------------------------------------------------ */
    environment {
        NAMESPACE         = 'payment-service1'
        APP_NAME         = 'payment-service'
        REGISTRY         = 'image-registry.openshift-image-registry.svc:5000'
        SEMANTIC_VERSION = "${params.SEMANTIC_VERSION ?: env.BUILD_NUMBER}"
        AUTH_SERVICE_URL = 'http://authentication-service:8080'
    }

    /* STAGES --------------------------------------------------------------- */
    stages {
        // 1) Checkout -------------------------------------------------------
        stage('Checkout') {
            steps {
                echo '📥  Checkout source…'
                checkout scm
            }
        }

        // 2) Build & Test ---------------------------------------------------
        stage('Build & Test') {
            steps {
                echo '🔨  Gradle build + test…'
                script {
                    sh 'chmod +x ./gradlew'
                    sh './gradlew clean build'
                }
            }
        }

        // 3) BuildConfig / ImageStream / Build ------------------------------
        stage('Build with OpenShift BuildConfig') {
            steps {
                script {
                    echo "🚀  Trigger build for ${APP_NAME}:${SEMANTIC_VERSION}"

                    sh """
                        oc project ${NAMESPACE}

                        # BuildConfig -------------------------------------------------
                        oc apply -f - <<EOF
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: ${APP_NAME}
  labels: { 
    app: ${APP_NAME},
    version: "1.0.0"
  }
spec:
  source: { type: Binary }
  strategy:
    type: Docker
    dockerStrategy: { dockerfilePath: Dockerfile }
  output:
    to: { kind: ImageStreamTag, name: ${APP_NAME}:latest }
  triggers: [ { type: Manual } ]
EOF

                        # ImageStream -----------------------------------------------
                        oc apply -f - <<EOF
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: ${APP_NAME}
  labels: { 
    app: ${APP_NAME},
    version: "1.0.0"
  }
spec: { lookupPolicy: { local: false } }
EOF

                        # Start build ----------------------------------------------
                        echo '⏳  oc start-build …'
                        oc start-build ${APP_NAME} --from-dir=. --wait --follow | cat

                        # Tag semver -----------------------------------------------
                        oc tag ${APP_NAME}:latest ${APP_NAME}:${SEMANTIC_VERSION}
                        echo '✅  Build finished.'
                    """
                }
            }
        }

        // 4) Apply manifests (inline) --------------------------------------
        stage('Apply OpenShift Resources') {
            steps {
                script {
                    echo '📜  Applying inline manifests…'
                    sh """
                        oc project ${NAMESPACE}

                        # ---------- Secret ----------
                        cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: ${APP_NAME}-secret
  labels: { 
    app: ${APP_NAME},
    version: "1.0.0"
  }
type: Opaque
stringData:
  DB_USERNAME: developer
  DB_PASSWORD: developer123
  DB_URL: jdbc:postgresql://one-gate-payment-db:5432/one_gate_payment
  JWT_SECRET: changeme
  AUTH_SERVICE_KEY: your-auth-service-key-here
EOF

                        # ---------- Service ----------
                        cat <<EOF | oc apply -f -
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}
  labels: { 
    app: ${APP_NAME},
    version: "1.0.0"
  }
spec:
  selector: { app: ${APP_NAME} }
  ports:
    - name: http
      port: 8080
      targetPort: 8080
  type: ClusterIP
EOF

                        # ---------- Deployment ----------
                        cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}
  labels: { 
    app: ${APP_NAME},
    version: "1.0.0"
  }
spec:
  replicas: 2
  selector: { matchLabels: { app: ${APP_NAME} } }
  template:
    metadata: { 
      labels: { 
        app: ${APP_NAME},
        version: "1.0.0"
      } 
    }
    spec:
      containers:
        - name: ${APP_NAME}
          image: ${REGISTRY}/${NAMESPACE}/${APP_NAME}:latest
          imagePullPolicy: Always
          ports: [ { containerPort: 8080 } ]
          env:
            - { name: DB_USERNAME, valueFrom: { secretKeyRef: { name: ${APP_NAME}-secret, key: DB_USERNAME } } }
            - { name: DB_PASSWORD, valueFrom: { secretKeyRef: { name: ${APP_NAME}-secret, key: DB_PASSWORD } } }
            - { name: DB_URL,     valueFrom: { secretKeyRef: { name: ${APP_NAME}-secret, key: DB_URL } } }
            - { name: JWT_SECRET, valueFrom: { secretKeyRef: { name: ${APP_NAME}-secret, key: JWT_SECRET } } }
            - { name: AUTH_SERVICE_URL, value: "\${AUTH_SERVICE_URL}" }
            - { name: AUTH_SERVICE_KEY, valueFrom: { secretKeyRef: { name: ${APP_NAME}-secret, key: AUTH_SERVICE_KEY } } }
          readinessProbe:
            httpGet: { path: /actuator/health, port: 8080 }
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
          livenessProbe:
            httpGet: { path: /actuator/health, port: 8080 }
            initialDelaySeconds: 60
            periodSeconds: 30
            timeoutSeconds: 10
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
EOF
                        echo '✅  Manifests applied.'
                    """
                }
            }
        }

        // 5) Deploy ---------------------------------------------------------
        stage('Deploy Application') {
            steps {
                script {
                    echo "🚢  Rollout ${APP_NAME}…"
                    sh """
                        oc project ${NAMESPACE}

                        # Set image => tag 'latest'
                        oc set image deployment/${APP_NAME} ${APP_NAME}=${REGISTRY}/${NAMESPACE}/${APP_NAME}:latest

                        # Restart rollout & wait
                        oc rollout restart deployment/${APP_NAME}
                        oc rollout status deployment/${APP_NAME} --timeout=300s | cat

                        # Info
                        oc get pods -l app=${APP_NAME}
                        oc get deployment ${APP_NAME} -o jsonpath='{.spec.template.spec.containers[0].image}'
                        echo ''
                    """
                }
            }
        }
    }

    /* POST ----------------------------------------------------------------- */
    post {
        success {
            echo """
🎉  SUCCESS  deployed ${APP_NAME}
Tags: latest, ${SEMANTIC_VERSION}
Namespace: ${NAMESPACE}
            """
        }
        failure {
            script {
                echo '❌  Pipeline FAILED.'
                sh """
                    echo '── Build logs (last 50 lines) ──'
                    oc logs -l build=${APP_NAME} --tail=50 -n ${NAMESPACE} || true
                """
            }
        }
    }
}