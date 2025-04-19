In this project, we’ll walk through creating a Jenkins CI/CD pipeline that automates cloning a GitHub repository into a remote production server via SSH, step-by-step.

📌 Step-by-Step Guide

✅ 1. Create a Git Repository

Start by pushing your project to a GitHub repository.

✅ 2. Add GitHub Credentials to Jenkins

Go to Manage Jenkins → Credentials and add your GitHub username & Personal Access Token (or password) as a credential. This enables Jenkins to interact with GitHub securely.

✅ 3. Setup SSH Access to the Remote Server

On your Jenkins server, generate an SSH key (if not already present):
# ssh-keygen -t rsa -b 4096

Copy the public key (id_rsa.pub) to your remote production server:
# ssh-copy-id root@192.168.150.131

Test it:
# ssh root@192.168.150.131

✅ 4. Add Jenkins SSH Private Key to Credential Manager
Add your SSH private key (id_rsa) to Jenkins credentials as SSH Username with private key.

✅ 5. Install Git on the Remote Server
Ensure Git is installed on your production server:
# yum install git 

✅ 6. Create the Jenkins Pipeline
Here’s a sample declarative pipeline:
pipeline {
    agent any

    environment {
        WORKER_NODE_IP = '192.168.150.131'
        GIT_REPO = 'https://github.com/tuhinmiazy73/Task-Tracker-Project.git'
        BRANCH = 'main'
        DEPLOY_DIR = '/var/www/html'
        BACKUP_DIR = '/var/www/html_backup'
    }

    stages {

        stage('SSH Login Test') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'JenkinsToWorkerSSH', keyFileVariable: 'SSH_KEY')]) {
                    sh '''
                        ssh -i $SSH_KEY -o StrictHostKeyChecking=no root@$WORKER_NODE_IP "echo '✅ SSH Login Successful'"
                    '''
                }
            }
        }

        stage('Backup Current Deployment') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'JenkinsToWorkerSSH', keyFileVariable: 'SSH_KEY')]) {
                    sh """
                        ssh -i \$SSH_KEY -o StrictHostKeyChecking=no root@${WORKER_NODE_IP} '
                            DEPLOY_DIR="${DEPLOY_DIR}"
                            BACKUP_DIR="${BACKUP_DIR}"
                            TIMESTAMP=\$(date +%Y%m%d_%H%M%S)
                            BACKUP_PATH="\$BACKUP_DIR/backup_\$TIMESTAMP"

                            mkdir -p "\$BACKUP_PATH"

                            if [ "\$(ls -A \$DEPLOY_DIR)" ]; then
                                mv \$DEPLOY_DIR/* "\$BACKUP_PATH/"
                                echo "📦 Backup completed to \$BACKUP_PATH"
                            else
                                echo "ℹ️ No files to backup"
                            fi
                        '
                    """
                }
            }
        }

        stage('Pull Latest Code') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'JenkinsToWorkerSSH', keyFileVariable: 'SSH_KEY')]) {
                    sh """
                        ssh -i \$SSH_KEY -o StrictHostKeyChecking=no root@${WORKER_NODE_IP} '
                            DEPLOY_DIR="${DEPLOY_DIR}"

                            if [ -d \$DEPLOY_DIR/.git ]; then
                                cd \$DEPLOY_DIR
                                git reset --hard
                                git pull origin ${BRANCH}
                                echo "🔄 Pulled latest code from GitHub"
                            else
                                echo "❌ Not a git repository. Please clone it first manually."
                                exit 1
                            fi
                        '
                    """
                }
            }
        }

        stage('Restart Apache Server') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'JenkinsToWorkerSSH', keyFileVariable: 'SSH_KEY')]) {
                    sh """
                        ssh -i \$SSH_KEY -o StrictHostKeyChecking=no root@${WORKER_NODE_IP} '
                            systemctl restart httpd && \
                            echo "🔁 Apache restarted successfully"
                        '
                    """
                }
            }
        }
    }

    post {
        failure {
            echo "❌ Deployment failed. Please check logs."
        }
        success {
            echo "✅ Deployment completed successfully."
        }
    }
}
