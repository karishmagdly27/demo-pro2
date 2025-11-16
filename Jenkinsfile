pipeline {
    agent any

    environment {
        APP_REPO      = 'https://github.com/karishmagdly27/demo-pro1.git'
        APP_NAME      = 'demo-app'
        DEV_HOST      = '3.134.102.126'
        STAGING_HOST  = '3.134.88.210'
        PROD_HOST     = '18.191.129.250'
        SONAR_HOST    = 'http://13.59.224.139:9000' // SonarQube server URL
    }

    stages {

        stage('Debug Branch') {
            steps {
                echo "BRANCH_NAME = ${env.BRANCH_NAME}"
            }
        }

        stage('Checkout') {
            steps {
                script {
                    def branch = env.BRANCH_NAME ?: 'dev'
                    echo "Checking out branch: ${branch}"
                    git branch: branch, url: "${APP_REPO}", credentialsId: 'git'
                }
            }
        }

        stage('SonarQube Analysis') {
            when {
                expression { return ['dev','stg','master'].contains(env.BRANCH_NAME) }
            }
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        sonar-scanner \
                        -Dsonar.projectKey=${APP_NAME}-${env.BRANCH_NAME} \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=${SONAR_HOST}
                    """
                }
            }
        }

        stage('Package') {
            steps {
                echo "Packaging application..."
                // Only include necessary app files
                sh "zip -r /tmp/${APP_NAME}.zip index.html css js images || true"
            }
        }

        stage('Upload to Nexus') {
            when {
                expression { return ['stg','master'].contains(env.BRANCH_NAME) }
            }
            steps {
                echo "Uploading artifact to Nexus..."
                nexusArtifactUploader artifacts: [[
                        artifactId: "${APP_NAME}",
                        classifier: '',
                        file: "/tmp/${APP_NAME}.zip",
                        type: 'zip'
                    ]],
                    credentialsId: 'nexus-creds',
                    groupId: 'com.demo',
                    nexusUrl: 'http://3.145.4.201:8081',
                    repository: 'demo-app-repo',
                    version: "${env.BRANCH_NAME}-${BUILD_NUMBER}"
            }
        }

        stage('Deploy') {
            when {
                expression { return ['dev','stg','master'].contains(env.BRANCH_NAME) }
            }
            steps {
                script {
                    def branchMap = [
                        'dev'   : env.DEV_HOST,
                        'stg'   : env.STAGING_HOST,
                        'master': env.PROD_HOST
                    ]
                    def host = branchMap[env.BRANCH_NAME]

                    if (env.BRANCH_NAME == 'master') {
                        input message: 'Approve deployment to PROD?', ok: 'Deploy'
                    }

                    deployToServer(host)
                }
            }
        }
    }

    post {
        always { echo 'Pipeline finished.' }
        success { echo "Build and deployment for ${env.BRANCH_NAME} succeeded." }
        failure { echo "Build or deployment failed." }
    }
}

// Function to deploy code to server
def deployToServer(host) {
    sshagent(['dev-app-server']) {
        sh """
        echo "Deploying ${APP_NAME} to ${host}..."
        ssh -o StrictHostKeyChecking=no ubuntu@${host} '
            sudo rm -rf /var/www/html/* &&
            sudo unzip -o /tmp/${APP_NAME}.zip -d /var/www/html &&
            sudo chown -R www-data:www-data /var/www/html &&
            sudo chmod -R 755 /var/www/html &&
            sudo systemctl reload nginx
        '
        """
    }
}
