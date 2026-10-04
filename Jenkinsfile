// Production pipeline only. Point the Jenkins job at this file
// (Script Path: Jenkinsfile.prod). Dev builds use Jenkinsfile.
pipeline {

    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 45, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '30'))
        skipDefaultCheckout(true)
    }

    environment {
        APP_NAME        = 'inspection-fe'
        ENVIRONMENT     = 'production'

        REPO_URL        = 'https://github.com/ankittyagi140/euroasiasci_inspection_FE.git'
        GIT_BRANCH      = 'release'

        DOCKER_BUILDKIT = '1'
        DOCKERFILE      = 'Dockerfile.prod'
        FE_APP_ENV      = 'production'
        FE_API_BASE_URL = 'https://inspection.api.euroasiasci.com/api/v1'

        REGISTRY        = 'ghcr.io/euroasiasci'
        IMAGE           = "${REGISTRY}/${APP_NAME}"
        // Prefixed so a dev job with the same build number cannot overwrite this image.
        IMAGE_TAG       = "prod-${BUILD_NUMBER}"

        PLATFORM_JOB    = 'EUROASIA_SCI_CLIENT_PORTAL/Euroasia_Inspection/Euroasia_inspection_Infra/Euroasia_inspection_Infra'
    }

    stages {

        stage('Checkout') {
            steps {
                git(
                    branch: "${GIT_BRANCH}",
                    credentialsId: 'github-pat',
                    url: "${REPO_URL}"
                )
                script {
                    env.GIT_SHA = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                set -eux
                npm ci
                '''
            }
        }

        stage('Lint') {
            steps {
                sh '''
                set -eux
                npm run lint
                npm run typecheck
                '''
            }
        }

        stage('Unit Tests') {
            steps {
                sh '''
                set -eux
                npm test -- --watch=false
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                set -eux
                docker build \
                    --pull \
                    --build-arg BUILDKIT_INLINE_CACHE=1 \
                    --build-arg APP_ENV=${FE_APP_ENV} \
                    --build-arg API_BASE_URL=${FE_API_BASE_URL} \
                    --build-arg APP_RELEASE=${GIT_SHA} \
                    --label org.opencontainers.image.version=${BUILD_NUMBER} \
                    --label org.opencontainers.image.revision=${GIT_SHA} \
                    --label org.opencontainers.image.source=${REPO_URL} \
                    -f ${DOCKERFILE} \
                    -t ${IMAGE}:${IMAGE_TAG} \
                    -t ${IMAGE}:${ENVIRONMENT} \
                    .
                '''
            }
        }

        stage('Login GHCR') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'ghcr-token',
                        usernameVariable: 'GHCR_USER',
                        passwordVariable: 'GHCR_TOKEN'
                    )
                ]) {
                    sh '''
                    echo "${GHCR_TOKEN}" | docker login ghcr.io \
                        --username "${GHCR_USER}" \
                        --password-stdin
                    '''
                }
            }
        }

        stage('Push Image') {
            steps {
                sh '''
                set -eux
                docker push ${IMAGE}:${IMAGE_TAG}
                docker push ${IMAGE}:${ENVIRONMENT}
                '''
            }
        }

        stage('Trigger Platform Deployment') {
            steps {
                script {
                    echo '========================================='
                    echo 'Triggering Platform Deployment...'
                    echo "Application : ${APP_NAME}"
                    echo "Environment : ${ENVIRONMENT}"
                    echo "Image Tag   : ${IMAGE_TAG}"
                    echo '========================================='

                    build(
                        job: "${PLATFORM_JOB}",
                        wait: true,
                        propagate: true,
                        parameters: [
                            string(name: 'APPLICATION', value: "${APP_NAME}"),
                            string(name: 'ENVIRONMENT', value: "${ENVIRONMENT}"),
                            string(name: 'IMAGE_TAG', value: "${IMAGE_TAG}"),
                            booleanParam(name: 'DRY_RUN', value: false),
                            booleanParam(name: 'SKIP_HEALTH_CHECK', value: false),
                        ]
                    )
                }
            }
        }
    }

    post {
        success {
            echo '''
=========================================
Frontend CI Successful
Docker image published
Platform deployment completed
=========================================
'''
        }
        failure {
            echo '''
=========================================
Frontend CI Failed
Check Platform Deployment logs if the
failure occurred after the image push.
=========================================
'''
        }
        always {
            sh 'docker logout ghcr.io || true'
            cleanWs()
        }
    }
}
