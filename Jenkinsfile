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

        stage('Checkout') {
            steps {
                echo "Checking out branch: ${env.BRANCH_NAME}"
                git branch: "${env.BRANCH_NAME}", url: "${APP_REPO}", credentialsId: 'github-token'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') { // Jenkins SonarQube server configurations
                    sh """
                        sonar-scanner \
                        -Dsonar.projectKey=${APP_NAME}-${BRANCH_NAME} \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=${SONAR_HOST}
                    """
                }
            }
        }

        stage('Package') {
            steps {
                echo "Packaging application..."
                sh "zip -r ${APP_NAME}.zip ."
            }
        }

        stage('Upload to Nexus') {
            when {
                expression {
                    // Only staging (stg) and prod (master) branches upload artifact
                    return ['stg','master'].contains(env.BRANCH_NAME)
                }
            }
            steps {
                echo "Uploading artifact to Nexus..."
                nexusArtifactUploader artifacts: [[
                        artifactId: "${APP_NAME}",
                        classifier: '',
                        file: "${APP_NAME}.zip",
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
                expression {
                    // Only deploy if branch is dev/stg/master
                    return ['dev','stg','master'].contains(env.BRANCH_NAME)
                }
            }
            steps {
                script {
                    // Map branches to environment hosts
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
        always {
            echo 'Pipeline finished.'
        }
        success {
            echo "Build and deployment for ${env.BRANCH_NAME} succeeded."
        }
        failure {
            echo "Build or deployment failed."
        }
    }
}

// Function to deploy code to server
def deployToServer(host) {
    sshagent(['dev-app-server']) {
        sh """
        ssh -o StrictHostKeyChecking=no ubuntu@${host} "sudo rm -rf /var/www/html/*"
        scp -o StrictHostKeyChecking=no -r * ubuntu@${host}:/var/www/html/
        ssh -o StrictHostKeyChecking=no ubuntu@${host} "sudo chown -R www-data:www-data /var/www/html"
        ssh -o StrictHostKeyChecking=no ubuntu@${host} "sudo systemctl reload nginx"
        """
    }
}
