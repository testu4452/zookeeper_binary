pipeline {

    agent any

    parameters {
        string(
            name: 'ZK_VERSION',
            defaultValue: '3.9.6',
            description: 'ZooKeeper version to deploy'
        )
    }

    environment {
        REMOTE_DEPLOY_DIR = '/opt/zookeeper'
        BACKUP_DIR        = '/opt/zookeeper_backup'
        TMP_FILE          = '/tmp/zookeeper.tar.gz'

        ZK_BINARY_NAME = "apache-zookeeper-${params.ZK_VERSION}-bin.tar.gz"
        ZK_BINARY_URL  = "https://raw.githubusercontent.com/testu4452/zookeeper_binary/main/apache-zookeeper-${params.ZK_VERSION}-bin.tar.gz"
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timestamps()
    }

    stages {

        stage('Validate Binary URL') {
            steps {
                sh '''
                    set -eu

                    echo "Binary : ${ZK_BINARY_NAME}"
                    echo "URL    : ${ZK_BINARY_URL}"

                    curl -f -L -I "${ZK_BINARY_URL}" >/dev/null

                    echo "✅ Binary URL verified."
                '''
            }
        }

        stage('Detect Version And Upgrade') {

            steps {

                script {

                    def targetVersion = params.ZK_VERSION

                    echo "🎯 Target Version: ${targetVersion}"

                    def currentVersion = sh(
                        script: """
                            if [ -x "${REMOTE_DEPLOY_DIR}/bin/zkServer.sh" ]; then
                                "${REMOTE_DEPLOY_DIR}/bin/zkServer.sh" version 2>&1 | \
                                grep -Eo '[0-9]+\\.[0-9]+\\.[0-9]+' | \
                                head -1
                            else
                                echo NOT_INSTALLED
                            fi
                        """,
                        returnStdout: true
                    ).trim()

                    boolean upgradeRequired = false

                    if (!currentVersion || currentVersion == 'NOT_INSTALLED') {

                        echo 'ℹ️ ZooKeeper not installed.'
                        upgradeRequired = true

                    } else {

                        echo "📌 Installed Version: ${currentVersion}"

                        def result = sh(
                            script: """
                                CURRENT="${currentVersion}"
                                TARGET="${targetVersion}"

                                if [ "\${CURRENT}" = "\${TARGET}" ]; then
                                    echo SKIP
                                elif [ "\$(printf '%s\\n' "\${CURRENT}" "\${TARGET}" | sort -V | tail -1)" = "\${TARGET}" ]; then
                                    echo UPGRADE
                                else
                                    echo SKIP
                                fi
                            """,
                            returnStdout: true
                        ).trim()

                        if (result == 'UPGRADE') {
                            upgradeRequired = true
                        }
                    }

                    if (!upgradeRequired) {
                        echo "✅ Same or newer version already installed. Skipping upgrade."
                        return
                    }

                    echo "⬆️ Upgrade required."

                    try {

                        sh '''
                            set -eu

                            sudo rm -rf "${BACKUP_DIR}"

                            if [ -d "${REMOTE_DEPLOY_DIR}" ]; then
                                sudo cp -a "${REMOTE_DEPLOY_DIR}" "${BACKUP_DIR}"
                                echo "✅ Backup created."
                            else
                                echo "ℹ️ No existing installation found."
                            fi
                        '''

                        sh '''
                            set -eu

                            echo "📥 Downloading ${ZK_BINARY_NAME}"

                            curl -f -L \
                                -o "${TMP_FILE}" \
                                "${ZK_BINARY_URL}"

                            test -s "${TMP_FILE}"

                            sudo mkdir -p "${REMOTE_DEPLOY_DIR}"

                            sudo rm -rf "${REMOTE_DEPLOY_DIR:?}"/*

                            sudo tar -xzf "${TMP_FILE}" \
                                -C "${REMOTE_DEPLOY_DIR}" \
                                --strip-components=1

                            sudo chown -R $(whoami):$(whoami) "${REMOTE_DEPLOY_DIR}"

                            chmod +x "${REMOTE_DEPLOY_DIR}/bin/"*.sh || true

                            echo "✅ Upgrade completed."
                        '''

                        sh '''
                            set -eu

                            test -f "${REMOTE_DEPLOY_DIR}/bin/zkServer.sh"

                            echo "🔎 Validating installation..."

                            "${REMOTE_DEPLOY_DIR}/bin/zkServer.sh" version

                            echo "✅ Validation successful."
                        '''

                        sh '''
                            set -eu

                            if [ -d "${BACKUP_DIR}" ]; then
                                sudo rm -rf "${BACKUP_DIR}"
                                echo "✅ Backup removed."
                            fi
                        '''

                        echo "🎉 ZooKeeper upgraded successfully to ${targetVersion}"

                    } catch (Exception ex) {

                        echo "❌ Upgrade failed."

                        sh '''
                            set +e

                            if [ -d "${BACKUP_DIR}" ]; then

                                sudo rm -rf "${REMOTE_DEPLOY_DIR}"

                                sudo mv "${BACKUP_DIR}" "${REMOTE_DEPLOY_DIR}"

                                echo "✅ Rollback successful."

                            else

                                echo "❌ Backup not found."

                            fi
                        '''

                        throw ex
                    }
                }
            }
        }
    }

    post {

        success {
            echo '✅ Pipeline completed successfully.'
        }

        failure {
            echo '❌ Pipeline failed.'
        }

        always {

            sh '''
                rm -f "${TMP_FILE}" 2>/dev/null || true
            '''

            cleanWs(deleteDirs: true)
        }
    }
}
