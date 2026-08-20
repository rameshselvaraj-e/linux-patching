// ============================================================
// Jenkins Pipeline: Linux Patching Automation
// ============================================================
// This pipeline applies patches to Ubuntu and RedHat servers
// using Ansible playbooks. It includes pre-checks, patching,
// post-validation, and report archiving.
//
// Prerequisites:
//   - Jenkins with Ansible plugin installed
//   - Ansible installed on the Jenkins agent
//   - SSH key credential stored in Jenkins (ID: ansible-ssh-key)
//   - Vault password credential if using Ansible Vault (ID: ansible-vault-pass)
//   - Target hosts reachable from Jenkins agent
// ============================================================

pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 2, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '30'))
        disableConcurrentBuilds()
        ansiColor('xterm')
    }

    // ==========================================================
    // PIPELINE PARAMETERS
    // ==========================================================
    parameters {
        choice(
            name: 'TARGET_GROUP',
            choices: ['all', 'ubuntu', 'redhat', 'dev', 'prod'],
            description: 'Ansible inventory group to patch'
        )
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'staging', 'prod'],
            description: 'Environment being patched (affects approval gates)'
        )
        booleanParam(
            name: 'AUTO_REBOOT',
            defaultValue: false,
            description: 'Automatically reboot servers if required after patching'
        )
        string(
            name: 'SERIAL_BATCH',
            defaultValue: '1',
            description: 'Number of hosts to patch simultaneously (1 = one at a time)'
        )
        booleanParam(
            name: 'DRY_RUN',
            defaultValue: false,
            description: 'Run Ansible in check mode (no changes applied)'
        )
        booleanParam(
            name: 'SKIP_APPROVAL',
            defaultValue: false,
            description: 'Skip manual approval gate (use for non-prod only)'
        )
        string(
            name: 'EXTRA_ANSIBLE_ARGS',
            defaultValue: '',
            description: 'Additional ansible-playbook arguments (e.g., -v, --tags security)'
        )
        string(
            name: 'ANSIBLE_VERSION',
            defaultValue: '',
            description: 'Ansible version to use (leave empty for system default, e.g. 2.16)'
        )
    }

    environment {
        ANSIBLE_CONFIG = "${WORKSPACE}/linux-patching/ansible.cfg"
        REPORTS_DIR    = "${WORKSPACE}/linux-patching/reports"
        PLAYBOOK_DIR   = "${WORKSPACE}/linux-patching/playbooks"
    }

    stages {

        // ======================================================
        // STAGE 1: CHECKOUT
        // ======================================================
        stage('Checkout') {
            steps {
                script {
                    echo "==> Checking out Linux patching repository..."
                }
                checkout scm
            }
        }

        // ======================================================
        // STAGE 2: SETUP ANSIBLE
        // ======================================================
        stage('Setup Ansible') {
            steps {
                script {
                    echo "==> Setting up Ansible environment..."

                    // Install Ansible if not present
                    sh '''
                        if ! command -v ansible-playbook &> /dev/null; then
                            echo "Ansible not found. Installing..."
                            pip3 install ansible
                        fi
                        ansible-playbook --version
                    '''

                    // Install specific version if requested
                    if (params.ANSIBLE_VERSION?.trim()) {
                        sh "pip3 install ansible==${params.ANSIBLE_VERSION}"
                    }

                    // Create reports directory
                    sh "mkdir -p ${REPORTS_DIR}"
                }
            }
        }

        // ======================================================
        // STAGE 3: SYNTAX CHECK
        // ======================================================
        stage('Syntax Check') {
            steps {
                script {
                    echo "==> Running Ansible syntax check..."
                }
                sh '''
                    ansible-playbook ${PLAYBOOK_DIR}/pre_patch.yml --syntax-check
                    ansible-playbook ${PLAYBOOK_DIR}/patch_linux.yml --syntax-check
                    ansible-playbook ${PLAYBOOK_DIR}/post_validate.yml --syntax-check
                '''
            }
        }

        // ======================================================
        // STAGE 4: CONNECTIVITY CHECK
        // ======================================================
        stage('Connectivity Check') {
            steps {
                script {
                    echo "==> Checking SSH connectivity to target hosts..."
                }
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ansible-ssh-key',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    sh """
                        export ANSIBLE_PRIVATE_KEY_FILE=\${SSH_KEY}
                        ansible ${params.TARGET_GROUP} -i ${WORKSPACE}/linux-patching/inventory/hosts.yml \
                            -u \${SSH_USER} -m ping
                    """
                }
            }
        }

        // ======================================================
        // STAGE 5: PRE-PATCH CHECKS
        // ======================================================
        stage('Pre-Patch Checks') {
            steps {
                script {
                    echo "==> Running pre-patch checks (disk, services, locks, packages)..."
                }
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ansible-ssh-key',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    sh """
                        export ANSIBLE_PRIVATE_KEY_FILE=\${SSH_KEY}
                        ansible-playbook ${PLAYBOOK_DIR}/pre_patch.yml \
                            -i ${WORKSPACE}/linux-patching/inventory/hosts.yml \
                            -u \${SSH_USER} \
                            -e "target_group=${params.TARGET_GROUP}" \
                            -e "serial_batch=${params.SERIAL_BATCH}" \
                            ${params.DRY_RUN ? '--check' : ''} \
                            ${params.EXTRA_ANSIBLE_ARGS}
                    """
                }
            }
        }

        // ======================================================
        // STAGE 6: APPROVAL GATE (PRODUCTION)
        // ======================================================
        stage('Approval Gate') {
            when {
                expression {
                    params.ENVIRONMENT == 'prod' && !params.SKIP_APPROVAL
                }
            }
            steps {
                script {
                    echo "==> Production patching requires manual approval..."
                    timeout(time: 30, unit: 'MINUTES') {
                        input(
                            message: "Approve patching for ${params.ENVIRONMENT} (${params.TARGET_GROUP})?",
                            ok: "Approve Patching",
                            submitter: 'patch-admins',
                            parameters: [
                                string(
                                    name: 'APPROVAL_REASON',
                                    defaultValue: '',
                                    description: 'Provide a reason/ticket number for this patching'
                                )
                            ]
                        )
                    }
                }
            }
        }

        // ======================================================
        // STAGE 7: APPLY PATCHES
        // ======================================================
        stage('Apply Patches') {
            steps {
                script {
                    echo "==> Applying patches to ${params.TARGET_GROUP}..."
                }
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ansible-ssh-key',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    sh """
                        export ANSIBLE_PRIVATE_KEY_FILE=\${SSH_KEY}
                        ansible-playbook ${PLAYBOOK_DIR}/patch_linux.yml \
                            -i ${WORKSPACE}/linux-patching/inventory/hosts.yml \
                            -u \${SSH_USER} \
                            -e "target_group=${params.TARGET_GROUP}" \
                            -e "serial_batch=${params.SERIAL_BATCH}" \
                            -e "auto_reboot=${params.AUTO_REBOOT}" \
                            ${params.DRY_RUN ? '--check' : ''} \
                            ${params.EXTRA_ANSIBLE_ARGS}
                    """
                }
            }
        }

        // ======================================================
        // STAGE 8: POST-PATCH VALIDATION
        // ======================================================
        stage('Post-Patch Validation') {
            steps {
                script {
                    echo "==> Running post-patch validation..."
                }
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ansible-ssh-key',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    sh """
                        export ANSIBLE_PRIVATE_KEY_FILE=\${SSH_KEY}
                        ansible-playbook ${PLAYBOOK_DIR}/post_validate.yml \
                            -i ${WORKSPACE}/linux-patching/inventory/hosts.yml \
                            -u \${SSH_USER} \
                            -e "target_group=${params.TARGET_GROUP}" \
                            -e "serial_batch=${params.SERIAL_BATCH}" \
                            ${params.EXTRA_ANSIBLE_ARGS}
                    """
                }
            }
        }
    }

    // ==========================================================
    // POST ACTIONS
    // ==========================================================
    post {
        always {
            script {
                echo "==> Archiving patch reports..."
            }
            // Archive all reports generated during the pipeline run
            archiveArtifacts(
                artifacts: 'linux-patching/reports/**',
                allowEmptyArchive: true,
                fingerprint: true
            )

            // Publish reports for easy viewing
            publishHTML([
                reportDir: 'linux-patching/reports',
                reportFiles: '*.txt',
                reportName: 'Patch Reports',
                keepAll: true,
                allowMissing: true
            ])
        }

        success {
            echo "==> SUCCESS: Linux patching pipeline completed successfully for ${params.TARGET_GROUP}"
        }

        failure {
            echo "==> FAILURE: Linux patching pipeline failed. Check logs and reports."
        }

        unstable {
            echo "==> UNSTABLE: Patching completed with warnings. Review reports."
        }

        cleanup {
            echo "==> Cleaning up workspace..."
            sh "rm -rf ${REPORTS_DIR}/*.tmp 2>/dev/null || true"
        }
    }
}

