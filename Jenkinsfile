pipeline {
    agent { label 'jenkins-agent' }

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        APP_NAME = "register-app"
        RELEASE = "1.0.0"
        DOCKER_CREDENTIALS = credentials("DOCKERHUB_CREDENTIALS")
        IMAGE_NAME = "chiomavee/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}" // Consider adding commit hash: "${RELEASE}-${BUILD_NUMBER}-${GIT_COMMIT}"
        //JENKINS_API_TOKEN = credentials("jenkins-api-token") // Remove this! Use better CD trigger
    }

    stages {
        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/chiomanwanedo/register-app.git'
            }
        }

        stage("Build Application") {
            steps {
                sh "set -e; mvn clean package" // Add set -e for error handling
            }
        }

        stage("Test Application") {
            steps {
                sh "set -e; mvn test" // Add set -e for error handling
            }
        }

        stage("SonarQube Analysis") {
            steps {
                script {
                    withSonarQubeEnv('SonarScanner') {
                        sh "set -e; mvn sonar:sonar" // Add set -e for error handling
                    }
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    // Build the Docker image
                    def app = docker.build("${IMAGE_NAME}:${env.BUILD_NUMBER}")

                    // Authenticate with Docker Hub and push the image
                    docker.withRegistry('https://index.docker.io/v1/', 'DOCKERHUB_CREDENTIALS') {
                        app.push('latest')
                        app.push("${env.BUILD_NUMBER}")
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                script {
                    sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ${IMAGE_NAME}:latest --no-progress --scanners vuln --exit-code 0 --severity HIGH,CRITICAL,MEDIUM --format table" // Include MEDIUM severity
                }
            }
        }

        stage("Cleanup Artifacts") {
            steps {
                script {
                    sh "docker image inspect ${IMAGE_NAME}:${env.BUILD_NUMBER} > /dev/null 2>&1 && docker rmi ${IMAGE_NAME}:${env.BUILD_NUMBER} || true" // Check if image exists before removing
                    sh "docker image inspect ${IMAGE_NAME}:latest > /dev/null 2>&1 && docker rmi ${IMAGE_NAME}:latest || true" // Check if image exists before removing
                }
            }
        }

        stage("Trigger CD Pipeline") {
            steps {
                // Use Jenkins' built-in mechanisms for triggering builds securely
                build job: 'gitops-register-app-cd', parameters: [string(name: 'IMAGE_TAG', value: "${IMAGE_TAG}")]
            }
        }
    }

    post {
        failure {
            emailext body: '''${SCRIPT, template="groovy-html.template"}''',
                      subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Failed",
                      mimeType: 'text/html', to: "vanessaegwuibe08@gmail.com"
        }
        success {
            emailext body: '''${SCRIPT, template="groovy-html.template"}''',
                      subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Successful",
                      mimeType: 'text/html', to: "vanessaegwuibe08@gmail.com"
        }
    }
}