
def run(cmd) {
    if (isUnix()) {
        sh cmd
    } else {
        bat cmd
    }
}

def ecrLogin() {
    withCredentials([
        string(credentialsId: 'aws-key-id', variable: 'AWS_ACCESS_KEY_ID'),
        string(credentialsId: 'aws-secret', variable: 'AWS_SECRET_ACCESS_KEY')
    ]) {
        run("""
            docker run --rm ^
            -e AWS_ACCESS_KEY_ID ^
            -e AWS_SECRET_ACCESS_KEY ^
            -e AWS_DEFAULT_REGION=${env.REGION} ^
            amazon/aws-cli:latest ^
            ecr get-login-password --region ${env.REGION} ^
            > ecr-password.txt
        """)

        run("""
            docker login --username AWS --password-stdin ${env.REGISTRY} < ecr-password.txt
        """)

        run("del ecr-password.txt")
    }
}

def deploy() {
    withCredentials([
        string(credentialsId: 'aws-key-id', variable: 'AWS_ACCESS_KEY_ID'),
        string(credentialsId: 'aws-secret', variable: 'AWS_SECRET_ACCESS_KEY')
    ]) {
        run("""
            docker run --rm ^
            -e AWS_ACCESS_KEY_ID ^
            -e AWS_SECRET_ACCESS_KEY ^
            -e AWS_DEFAULT_REGION=${env.REGION} ^
            -v "%WORKSPACE%:/workspace" ^
            -w /workspace ^
            amazon/aws-cli:latest ^
            ssm send-command ^
            --region ${env.REGION} ^
            --instance-ids ${env.INSTANCE} ^
            --document-name AWS-RunShellScript ^
            --parameters file://deploy.json
        """)
    }
}

pipeline {
    agent any

    environment {
        REGION   = 'ap-south-1'
        ACCOUNT  = '116948313314'
        REPO     = 'hello-jenkins'
        INSTANCE = 'none'

        REGISTRY = "${ACCOUNT}.dkr.ecr.${REGION}.amazonaws.com"
        IMG      = "${REPO}:${BUILD_NUMBER}"
        REMOTE   = "${REGISTRY}/${REPO}:latest"
    }

    stages {

        stage('Build') {
            steps {
                script {
                    run "docker build -t ${IMG} ."
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    run "docker run --rm ${IMG} nginx -t"
                }
            }
        }

        stage('Push') {
            steps {
                script {
                    ecrLogin()

                    run "docker tag ${IMG} ${REMOTE}"

                    run "docker push ${REMOTE}"
                }
            }
        }

        stage('Deploy') {
            when {
                not {
                    environment name: 'INSTANCE', value: 'none'
                }
            }

            steps {
                script {
                    deploy()
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed. Check Console Output.'
        }

        always {
            echo 'Jenkins pipeline execution finished.'
        }
    }
}
