pipeline {
    agent any

    tools {
        jdk 'jdk21'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage("Clean Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Git Checkout") {
            steps {
                git branch: 'master', url: 'https://github.com/Mahesh-ai-ux/amazon-Devsecops-main.git'
            }
        }

        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=amazon \
                    -Dsonar.projectKey=amazon
                    '''
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    timeout(time: 3, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                    }
                }
            }
        }

        stage("Install NPM Dependencies") {
            steps {
                sh "npm install"
            }
        }

        stage("Trivy File Scan") {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }

        stage("Build Docker Image") {
            steps {
                script {
                    env.IMAGE_TAG = "mahesh9030/amazon:${BUILD_NUMBER}"

                    sh "docker rmi -f amazon ${env.IMAGE_TAG} || true"

                    sh "docker build -t amazon ."
                }
            }
        }

        stage("Tag & Push to DockerHub") {
            steps {
                script {
                    withCredentials([string(credentialsId: 'docker-cred', variable: 'dockerpwd')]) {

                        sh "docker login -u mahesh9030 -p ${dockerpwd}"

                        sh "docker tag amazon ${env.IMAGE_TAG}"

                        sh "docker push ${env.IMAGE_TAG}"

                        sh "docker tag amazon mahesh9030/amazon:latest"

                        sh "docker push mahesh9030/amazon:latest"
                    }
                }
            }
        }

        stage("Trivy Scan Docker Image") {
            steps {
                script {
                    sh """
                    echo 'Running Trivy Scan on Docker Image'

                    trivy image -f json -o trivy-image.json ${env.IMAGE_TAG}

                    trivy image -f table -o trivy-image.txt ${env.IMAGE_TAG}
                    """
                }
            }
        }

        stage("Deploy Docker Container") {
            steps {
                script {
                    sh "docker rm -f amazon || true"

                    sh "docker run -d --name amazon -p 80:80 ${env.IMAGE_TAG}"
                }
            }
        }
    }

    post {
        always {
            script {

                def buildStatus = currentBuild.currentResult

                def buildUser = currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause')[0]?.userId ?: 'Github User'

                emailext(
                    subject: "Pipeline ${buildStatus}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",

                    body: """
                    <p>Amazon CI/CD Pipeline Status</p>

                    <p><b>Project:</b> ${env.JOB_NAME}</p>

                    <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>

                    <p><b>Status:</b> ${buildStatus}</p>

                    <p><b>Triggered By:</b> ${buildUser}</p>

                    <p><b>Build URL:</b>
                    <a href="${env.BUILD_URL}">
                    ${env.BUILD_URL}
                    </a></p>
                    """,

                    to: 'mareddy7619@gmail.com',

                    from: 'mareddy7619@gmail.com',

                    mimeType: 'text/html',

                    attachmentsPattern: 'trivyfs.txt,trivy-image.json,trivy-image.txt'
                )
            }
        }
    }
}