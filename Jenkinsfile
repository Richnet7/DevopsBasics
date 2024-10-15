pipeline {
    agent any
    environment {
        DOCKER_SERVER = '18.219.94.82'
        DOCKER_USER = 'ubuntu'
        DOCKER_HUB_REPO = 'richnet7/richnet'
        DOCKER_HUB_CREDENTIALS = 'dockerhub_credentials_id'
        IMAGE_TAG = "${env.BUILD_NUMBER}" // Dynamic tag based on build number
        SSH_CREDENTIALS_ID = 'ssh-credentials-id'
        REPO_URL = 'https://github.com/Richnet7/DevopsBasics.git'
    }
    tools {
        jdk 'myjava'
        maven 'mymaven'
    }
    stages {
        stage('Clone Repository') {
            steps {
                echo 'Cloning repository..'
                withCredentials([usernamePassword(credentialsId: 'Richnet7', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                    script {
                        // Remove existing repository clone if present
                        sh "rm -rf /var/lib/jenkins/workspace/cicd-pipeline/devops-basics"
                        // Clone repository
                        git credentialsId: 'Richnet7', url: env.REPO_URL, branch: 'master', dir: '/var/lib/jenkins/workspace/cicd-pipeline/devops-basics'
                    }
                }
            }
        }
        stage('Compile') {
            steps {
                echo 'Compiling..'
                sh 'mvn compile'
            }
        }
        stage('Package') {
            steps {
                echo 'Packaging..'
                sh 'mvn package'
            }
        }
        stage('Clear Docker Server') {
            steps {
                echo 'Clearing Docker Server..'
                script {
                    sshagent(credentials: [env.SSH_CREDENTIALS_ID]) {
                        def containerIds = sh(script: "ssh -o StrictHostKeyChecking=no ${env.DOCKER_USER}@${env.DOCKER_SERVER} 'docker ps -aq'", returnStdout: true).trim()
                        if (containerIds) {
                            sh "ssh -o StrictHostKeyChecking=no ${env.DOCKER_USER}@${env.DOCKER_SERVER} 'docker rm -f ${containerIds}'"
                        } else {
                            echo "No containers to remove."
                        }
                        sh "ssh -o StrictHostKeyChecking=no ${env.DOCKER_USER}@${env.DOCKER_SERVER} 'yes | docker system prune --all'"
                    }
                }
            }
        }
        stage('Copy WAR to Docker Server') {
            steps {
                echo 'Copying WAR to Docker Server..'
                sshagent(credentials: [env.SSH_CREDENTIALS_ID]) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ${env.DOCKER_USER}@${env.DOCKER_SERVER} 'rm -f /home/ubuntu/webapp.war'
                    scp -o StrictHostKeyChecking=no '/var/lib/jenkins/workspace/${env.JOB_NAME}/webapp/target/webapp.war' ${env.DOCKER_USER}@${env.DOCKER_SERVER}:/home/ubuntu/
                    """
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                echo 'Building Docker Image..'
                sshagent(credentials: [env.SSH_CREDENTIALS_ID]) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ${env.DOCKER_USER}@${env.DOCKER_SERVER} 'docker build -t ${env.DOCKER_HUB_REPO}:${env.IMAGE_TAG} /home/ubuntu'
                    """
                }
            }
        }
        stage('Push Docker Image') {
            steps {
                echo 'Pushing Docker Image..'
                sshagent(credentials: [env.SSH_CREDENTIALS_ID]) {
                    withCredentials([usernamePassword(credentialsId: env.DOCKER_HUB_CREDENTIALS, usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        sh """
                        ssh -o StrictHostKeyChecking=no ${env.DOCKER_USER}@${env.DOCKER_SERVER} 'echo ${DOCKER_PASSWORD} | docker login -u ${DOCKER_USERNAME} --password-stdin'
                        ssh -o StrictHostKeyChecking=no ${env.DOCKER_USER}@${env.DOCKER_SERVER} 'docker push ${env.DOCKER_HUB_REPO}:${env.IMAGE_TAG}'
                        """
                    }
                }
            }
        }
        stage('Run Docker Image') {
            steps {
                echo 'Running Docker Image..'
                sshagent(credentials: [env.SSH_CREDENTIALS_ID]) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ${env.DOCKER_USER}@${env.DOCKER_SERVER} 'sudo docker run -d --name our_app_container -p 8080:8080 ${env.DOCKER_HUB_REPO}:${env.IMAGE_TAG}'
                    """
                }
            }
        }
    }
    post {
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
