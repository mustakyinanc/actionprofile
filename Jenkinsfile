pipeline {
    agent any

    environment {
        ECR_REPOSITORY = 'actionprofile'
        AWS_REGION     = 'us-east-1'

        // Set to 'true' to intentionally fail the pipeline for demo purposes
        FORCE_FAIL     = 'false'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    script {
                        if (env.FORCE_FAIL == 'true') {
                            error('[DEMO] Build intentionally failed. FORCE_FAIL=true')
                        }
                    }
                    sh 'mvn clean install -DskipTests'
                    archiveArtifacts artifacts: 'target/*.war', fingerprint: true
                }
            }
        }

        stage('Unit Test') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    sh 'mvn test'
                }
            }
            post {
                always {
                    junit testResults: 'target/surefire-reports/*.xml',
                          allowEmptyResults: true
                }
            }
        }

        stage('Build Docker Image & Push to ECR') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    withCredentials([
                        string(credentialsId: 'AWS_ACCESS_KEY_ID',     variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        sh '''
                            export AWS_ACCESS_KEY_ID="$AWS_ACCESS_KEY_ID"
                            export AWS_SECRET_ACCESS_KEY="$AWS_SECRET_ACCESS_KEY"
                            export AWS_DEFAULT_REGION="$AWS_REGION"

                            ECR_REGISTRY=$(aws ecr describe-repositories \
                                --repository-names "$ECR_REPOSITORY" \
                                --region "$AWS_REGION" \
                                --query "repositories[0].repositoryUri" \
                                --output text | sed "s|/$ECR_REPOSITORY||")

                            IMAGE_TAG="${GIT_COMMIT:0:7}"
                            FULL_IMAGE="$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"

                            aws ecr get-login-password --region "$AWS_REGION" | \
                                docker login --username AWS --password-stdin "$ECR_REGISTRY"

                            docker build -t "$FULL_IMAGE" .
                            docker push "$FULL_IMAGE"
                            docker tag "$FULL_IMAGE" "$ECR_REGISTRY/$ECR_REPOSITORY:latest"
                            docker push "$ECR_REGISTRY/$ECR_REPOSITORY:latest"

                            echo "Image pushed: $FULL_IMAGE"
                        '''
                    }
                }
            }
        }

        stage('Notify DevOps Agent') {
            when {
                expression { currentBuild.currentResult == 'FAILURE' }
            }
            steps {
                withCredentials([
                    string(credentialsId: 'DEVOPS_AGENT_WEBHOOK_URL',    variable: 'DEVOPS_AGENT_WEBHOOK_URL'),
                    string(credentialsId: 'DEVOPS_AGENT_WEBHOOK_SECRET', variable: 'DEVOPS_AGENT_WEBHOOK_SECRET')
                ]) {
                    sh '''
                        TIMESTAMP=$(date -u +%Y-%m-%dT%H:%M:%S.000Z)
                        INCIDENT_ID=$(cat /proc/sys/kernel/random/uuid)

                        PAYLOAD=$(printf \
                            '{"eventType":"incident","incidentId":"%s","action":"created","priority":"HIGH","title":"[JENKINS] actionprofile pipeline failed on %s","description":"Source: Jenkins Pipeline. One or more stages failed in the actionprofile pipeline. Investigate the failure related to the latest commit. Job:%s Branch:%s Commit:%s Build URL:%s 1. Review changes in the latest commit 2. Analyze Jenkins stage logs 3. Identify root cause and suggest a fix","service":"actionprofile-jenkins","timestamp":"%s","data":{"metadata":{"source":"Jenkins","job":"%s","branch":"%s","commit":"%s","build_number":"%s","build_url":"%s"}}}' \
                            "$INCIDENT_ID" \
                            "${BRANCH_NAME:-main}" \
                            "${JOB_NAME}" \
                            "${BRANCH_NAME:-main}" \
                            "${GIT_COMMIT:-unknown}" \
                            "${BUILD_URL}" \
                            "$TIMESTAMP" \
                            "${JOB_NAME}" \
                            "${BRANCH_NAME:-main}" \
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

                        echo "DevOps Agent webhook -> HTTP $HTTP_STATUS"
                        cat /tmp/response.txt
                    '''
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
