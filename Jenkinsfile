pipeline {
    agent any
    environment {
        DOCKER_SERVER = '18.118.30.144'
        DOCKER_USER = 'ubuntu'
        DOCKER_HUB_REPO = 'richnet7/richnet:tagname'
        DOCKER_HUB_CREDENTIALS = 'dockerhub_credentials_id'
        IMAGE_TAG = "${env.BUILD_NUMBER}" // Dynamic tag based on build number
        SSH_CREDENTIALS_ID = 'ssh-credentials-id'
        REPO_URL = 'https://github.com/theitern/devops-basics.git'
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
                        sh "rm -rf ${env.WORKSPACE}/devops-basics"
                        git credentialsId: 'Richnet7', url: env.REPO_URL, branch: 'master', dir: "${env.WORKSPACE}/devops-basics"
                    }
                }
            }
        }
        stage('Compile and Package') {
            steps {
                echo 'Compiling and Packaging..'
                dir("${env.WORKSPACE}/devops-basics/webapp") {
                    sh 'mvn compile'
                    sh 'mvn package'
                }
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
                    script {
                        def warFileExists = sh(script: "ls -la /var/lib/jenkins/workspace/${env.JOB_NAME}/webapp/target/webapp.war", returnStatus: true) == 0
                        def targetDirExists = sh(script: "ls -la /var/lib/jenkins/workspace/${env.JOB_NAME}/webapp/target", returnStatus: true) == 0
                        if (!targetDirExists) {
                            error 'Target directory not found, aborting the pipeline.'
                        }
                        if (!warFileExists) {
                            error 'WAR file not found, aborting the pipeline.'
                        }

                        sh """
                        ssh -o StrictHostKeyChecking=no ${env.DOCKER_USER}@${env.DOCKER_SERVER} 'rm -f /home/ubuntu/webapp.war'
                        scp -o StrictHostKeyChecking=no /var/lib/jenkins/workspace/${env.JOB_NAME}/webapp/target/webapp.war ${env.DOCKER_USER}@${env.DOCKER_SERVER}:/home/ubuntu/
                        """
                    }
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                echo 'Building Docker Image..'
                sshagent(credentials: [env.SSH_CREDENTIALS_ID]) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ${env.DOCKER_USER}@${env.DOCKER_SERVER} 'docker build -t ${env.DOCKER_HUB_REPO}:${env.IMAGE_TAG} -f /home/ubuntu/Dockerfile /home/ubuntu'
                    """
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
            script {
                // Cleanup actions can be added here
                // e.g., stop and remove containers
            }
        }
    }
}
