pipeline {
    agent any
    parameters {
        string(name: 'VM_USER', defaultValue: 'ubuntu', description: 'SSH user for the VM')
        string(name: 'VM_HOST', defaultValue: '10.73.76.154', description: 'VM IP address')
        credentials(name: 'SSH_CREDENTIALS_ID', description: 'Jenkins SSH credentials ID for VM access')
    }
    stages {
        stage('Syntax Validation') {
            steps {
                echo 'Validating Jenkinsfile syntax...'
                // Jenkins validates syntax automatically when saving; this stage is for custom checks if needed
                sh '''
                # Validate that required parameters are non-empty
                if [ -z "${VM_USER}" ] || [ -z "${VM_HOST}" ] || [ -z "${SSH_CREDENTIALS_ID}" ]; then
                    echo "Error: VM_USER, VM_HOST, or SSH_CREDENTIALS_ID is empty"
                    exit 1
                fi
                echo 'Parameters validated.'
                '''
            }
        }
        stage('Logic Checks') {
            steps {
                echo 'Performing logic checks...'
                // Verify GitHub repository accessibility
                sh '''
                git ls-remote https://github.com/testu4452/zookeeper_binary.git >/dev/null 2>&1
                if [ $? -ne 0 ]; then
                    echo "Error: Cannot access GitHub repository testu4452/zookeeper_binary"
                    exit 1
                fi
                echo 'GitHub repository is accessible.'
                '''
            }
        }
        stage('Linting') {
            steps {
                echo 'Linting...'
                // Lint the Jenkinsfile using jenkinsfile-linter if available (optional)
                // We'll just check for presence of binary in repo after checkout
                sh '''
                echo 'No specific linting tool configured; proceeding to checkout.'
                '''
            }
        }
        stage('Checkout and Prepare') {
            steps {
                echo 'Checking out ZooKeeper binary repository...'
                git branch: 'main', url: 'https://github.com/testu4452/zookeeper_binary.git'
                echo 'Verifying binary exists...'
                sh '''
                if [ ! -f apache-zookeeper-3.8.7-bin.tar.gz ]; then
                    echo "Error: Binary apache-zookeeper-3.8.7-bin.tar.gz not found in repository"
                    exit 1
                fi
                echo 'Binary found.'
                '''
            }
        }
        stage('Deploy to VM') {
            steps {
                echo 'Deploying ZooKeeper binary to VM...'
                // Use SSH agent with credentials
                sshagent([credentials: "${SSH_CREDENTIALS_ID}""]) {
                    sh """
                    # Copy binary to VM
                    scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null apache-zookeeper-3.8.7-bin.tar.gz ${VM_USER}@${VM_HOST}:/tmp/
                    
                    # SSH into VM to install
                    ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${VM_USER}@${VM_HOST} <<'EOSSH'
                    set -e
                    echo 'Creating ZooKeeper directory...'
                    sudo mkdir -p /opt/zookeeper
                    echo 'Extracting binary...'
                    sudo tar -xzf /tmp/apache-zookeeper-3.8.7-bin.tar.gz -C /opt/zookeeper --strip-components=1
                    echo 'Setting permissions...'
                    sudo chown -R ${VM_USER}:${VM_USER} /opt/zookeeper
                    echo 'Cleaning up...'
                    rm -f /tmp/apache-zookeeper-3.8.7-bin.tar.gz
                    echo 'Deployment completed successfully.'
                    EOSSH
                    """
                }
            }
        }
        stage('Post-Deployment Verification') {
            steps {
                echo 'Verifying deployment...'
                sshagent([credentials: "${SSH_CREDENTIALS_ID}""]) {
                    sh """
                    ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${VM_USER}@${VM_HOST} <<'EOSSH'
                    set -e
                    if [ -d /opt/zookeeper/bin ] && [ -f /opt/zookeeper/bin/zkServer.sh ]; then
                        echo 'ZooKeeper binary directory structure verified.'
                        # Check version
                        /opt/zookeeper/bin/zkServer.sh -version
                    else
                        echo 'Error: ZooKeeper installation verification failed.'
                        exit 1
                    fi
                    EOSSH
                    """
                }
            }
        }
    }
    post {
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Check console output for details.'
        }
    }
}