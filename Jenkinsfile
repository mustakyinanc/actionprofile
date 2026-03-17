pipeline {
    agent any

    environment {
        // ECR
        ECR_REPOSITORY = 'vprofile-appimage'
        AWS_REGION     = 'us-east-1'

        // SonarQube - Jenkins > Manage Jenkins > Configure System'da
        // "SonarQube servers" bölümüne eklenmeli
        SONAR_SERVER   = 'SonarQube'          // Buradaki isim Jenkins'te tanımladığınız server adıyla eşleşmeli
        SONAR_SCANNER  = 'SonarScanner'       // Buradaki isim Jenkins'te tanımladığınız scanner adıyla eşleşmeli

        // Maven - Jenkins > Manage Jenkins > Global Tool Configuration'da
        // "Maven" bölümüne "Maven-3.9.9" adıyla eklenmeli
        MAVEN_VERSION  = 'Maven-3.9.9'
    }

    tools {
        maven "${MAVEN_VERSION}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean install -DskipTests'
                archiveArtifacts artifacts: 'target/*.war', fingerprint: true
            }
        }

        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit testResults: 'target/surefire-reports/*.xml',
                          allowEmptyResults: true
                }
            }
        }

        stage('Code Analysis - SonarQube') {
            steps {
                withSonarQubeEnv("${SONAR_SERVER}") {
                    sh '''
                        mvn sonar:sonar \
                            -Dsonar.projectKey=vprofile \
                            -Dsonar.projectName=vprofile \
                            -Dsonar.java.binaries=target/classes
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image & Push to ECR') {
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID',     variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                        export AWS_ACCESS_KEY_ID="$AWS_ACCESS_KEY_ID"
                        export AWS_SECRET_ACCESS_KEY="$AWS_SECRET_ACCESS_KEY"
                        export AWS_DEFAULT_REGION="$AWS_REGION"

                        # ECR registry URL'ini al
                        ECR_REGISTRY=$(aws ecr describe-repositories \
                            --repository-names "$ECR_REPOSITORY" \
                            --region "$AWS_REGION" \
                            --query "repositories[0].repositoryUri" \
                            --output text | sed "s|/$ECR_REPOSITORY||")

                        IMAGE_TAG="${GIT_COMMIT:0:7}"
                        FULL_IMAGE="$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"

                        # ECR'a login
                        aws ecr get-login-password --region "$AWS_REGION" | \
                            docker login --username AWS --password-stdin "$ECR_REGISTRY"

                        # Docker image build
                        docker build -t "$FULL_IMAGE" .

                        # ECR'a push
                        docker push "$FULL_IMAGE"

                        # latest tag da push et
                        docker tag "$FULL_IMAGE" "$ECR_REGISTRY/$ECR_REPOSITORY:latest"
                        docker push "$ECR_REGISTRY/$ECR_REPOSITORY:latest"

                        echo "Image pushed: $FULL_IMAGE"
                    '''
                }
            }
        }
    }

    post {
        failure {
            withCredentials([
                string(credentialsId: 'DEVOPS_AGENT_WEBHOOK_URL',    variable: 'DEVOPS_AGENT_WEBHOOK_URL'),
                string(credentialsId: 'DEVOPS_AGENT_WEBHOOK_SECRET', variable: 'DEVOPS_AGENT_WEBHOOK_SECRET')
            ]) {
                sh '''
                    TIMESTAMP=$(date -u +%Y-%m-%dT%H:%M:%S.000Z)
                    INCIDENT_ID=$(cat /proc/sys/kernel/random/uuid)

                    PAYLOAD=$(printf \
                        '{"eventType":"incident","incidentId":"%s","action":"created","priority":"HIGH","title":"[Vprofile CICD] Pipeline failed on %s","description":"One or more stages failed in the Vprofile CICD pipeline. Investigate the failure related to the latest commit and identify the root cause. Job:%s Branch:%s Commit:%s Build URL:%s 1. Review changes in the latest commit 2. Check SonarQube quality gate results 3. Analyze stage logs 4. Identify root cause and suggest a fix","service":"vprofile","timestamp":"%s","data":{"metadata":{"job":"%s","branch":"%s","commit":"%s","build_number":"%s","build_url":"%s"}}}' \
                        "$INCIDENT_ID" \
                        "${BRANCH_NAME:-unknown}" \
                        "${JOB_NAME}" \
                        "${BRANCH_NAME:-unknown}" \
                        "${GIT_COMMIT:-unknown}" \
                        "${BUILD_URL}" \
                        "$TIMESTAMP" \
                        "${JOB_NAME}" \
                        "${BRANCH_NAME:-unknown}" \
                        "${GIT_COMMIT:-unknown}" \
                        "${BUILD_NUMBER}" \
                        "${BUILD_URL}")

                    SIGNATURE=$(echo -n "${TIMESTAMP}:${PAYLOAD}" | \
                        openssl dgst -sha256 -hmac "$DEVOPS_AGENT_WEBHOOK_SECRET" -binary | \
                        base64 -w 0)

                    HTTP_STATUS=$(curl -s -o /tmp/response.txt -w "%{http_code}" \
                        -X POST "$DEVOPS_AGENT_WEBHOOK_URL" \
                        -H "Content-Type: application/json" \
                        -H "x-amzn-event-timestamp: $TIMESTAMP" \
                        -H "x-amzn-event-signature: $SIGNATURE" \
                        -d "$PAYLOAD")

                    echo "DevOps Agent webhook → HTTP $HTTP_STATUS"
                    cat /tmp/response.txt
                    exit 0
                '''
            }
        }

        always {
            cleanWs()
        }
    }
}
