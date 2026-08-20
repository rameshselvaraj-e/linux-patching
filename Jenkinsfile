pipeline {
  agent any

  environment {
        ANSIBLE_HOST = '10.0.10.10'
        ANSIBLE_USER = 'itadmin'
        PLAYBOOK_PATH = '/data/ansible/playbooks/test.yml'
        INVENTORY_PATH = '/data/ansible/inventory/hosts'
    }
  
  stages {
    stage('Hello') {
      steps {
        echo 'Hello World'
      }
    }
    stage('Build') {
      steps {
        echo 'Building'
      }
    }
    stage('Deploy') {
      steps {
        echo 'Deploying'
      }
    }

    stage('Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/rameshselvaraj-e/jenkinspipeline.git'
      }
    }
      
    stage('Copy Playbook to Ansible Server') {
      steps {
        sshagent(credentials: ['ansible-server-ssh']) {
          sh """
          scp -o StrictHostKeyChecking=no -r ./playbooks/* ${ANSIBLE_USER}@${ANSIBLE_HOST}:/data/ansible/playbooks/
          """
        }
      }
    }

     stage('Apply Patches') {
            steps {
                sshagent(credentials: ['ansible-server-ssh']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${ANSIBLE_USER}@${ANSIBLE_HOST} \
                        'ansible-playbook -i ${INVENTORY_PATH} ${PLAYBOOK_PATH} \
                        -e target_hosts=${params.TARGET_HOSTS} -e batch_size=${params.BATCH_SIZE}'
                    """
                }
            }
        }
    }
   post {
        success {
            echo 'Patching completed successfully.'
        }
        failure {
            echo 'Patching failed — check console output.'
        }
        always {
            archiveArtifacts artifacts: '**/*.log', allowEmptyArchive: true
        }
    }
}
