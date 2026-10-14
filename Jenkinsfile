pipeline {
    agent any

    environment {
        IMAGE_NAME = 'hospital-platform'
        IMAGE_TAG  = "v1.${BUILD_NUMBER}"
    }

    options {
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
    }

    triggers { pollSCM('H/2 * * * *') }

    stages {
        stage('SCM Checkout') {
            steps { checkout scm }
        }

        stage('Install Dependencies') {
            steps {
                bat '''
                    python --version
                    python -m venv venv
                    venv\\Scripts\\python -m pip install --upgrade pip
                    venv\\Scripts\\python -m pip install -r requirements.txt
                '''
            }
        }

        stage('Quality Gates') {
            parallel {
                stage('Lint (Flake8)') {
                    steps {
                        bat 'venv\\Scripts\\python -m flake8 app/ --max-line-length=120 --count --statistics'
                    }
                }
                stage('Unit & API Tests') {
                    steps {
                        bat '''
                            if not exist reports mkdir reports
                            venv\\Scripts\\python -m pytest -v --cov=app --cov-fail-under=80 --junitxml=reports/test-results.xml
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                bat '''
                    docker build -t %IMAGE_NAME%:%IMAGE_TAG% .
                    docker tag %IMAGE_NAME%:%IMAGE_TAG% %IMAGE_NAME%:latest
                '''
            }
        }

        stage('Image Security Scan (Trivy)') {
            steps {
                bat '''
                    where trivy >nul 2>&1
                    if %ERRORLEVEL% EQU 0 (
                        trivy image --severity HIGH,CRITICAL --exit-code 0 %IMAGE_NAME%:%IMAGE_TAG%
                    ) else (
                        echo Trivy not installed - skipping scan
                    )
                '''
            }
        }

        stage('Deploy to Minikube') {
            steps {
                bat '''
                    minikube image load %IMAGE_NAME%:%IMAGE_TAG%
                    kubectl apply -f k8s/
                    kubectl set image deployment/hospital-api api=%IMAGE_NAME%:%IMAGE_TAG%
                    kubectl rollout status deployment/hospital-api --timeout=120s
                '''
            }
        }
    }
    post {
        always {
            junit allowEmptyResults: true, testResults: 'reports/test-results.xml'
            cleanWs deleteDirs: true, notFailBuild: true
        }
        success { echo "Build #${BUILD_NUMBER} deployed as ${IMAGE_NAME}:${IMAGE_TAG}" }
        failure { echo "Build #${BUILD_NUMBER} FAILED - check console output" }
    }
}
