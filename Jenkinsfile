pipeline {
  agent any

  environment {
    GIT_BRANCH       = "dev"
    ANSIBLE_HOST     = "4.145.84.26"
    DESTROY_MODE     = "false"
    VENV_PATH        = "/home/boho/ansible-env"
    WORKDIR          = "/tmp/ansible-key-auto-run"
    REPO_URL         = "https://github.com/kitsanaphon1/ansible-key-auto.git"
  }

  stages {

    stage('📥 Checkout Jenkinsfile') {
      steps {
        checkout scm
      }
    }

    stage('☁️ Provision หรือ Destroy VM') {
      steps {
        withCredentials([
          sshUserPrivateKey(credentialsId: 'ssh-ansible-agent', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER'),
          string(credentialsId: 'AZURE_CLIENT_ID',       variable: 'AZURE_CLIENT_ID'),
          string(credentialsId: 'AZURE_SECRET',          variable: 'AZURE_SECRET'),
          string(credentialsId: 'AZURE_TENANT',          variable: 'AZURE_TENANT'),
          string(credentialsId: 'AZURE_SUBSCRIPTION_ID', variable: 'AZURE_SUBSCRIPTION_ID')
        ]) {
          script {
            def playbook = (DESTROY_MODE == "true") ? "destroy-linux-vm.yaml" : "create-linux-vm.yaml"

            sh """
              ssh -i $SSH_KEY -o StrictHostKeyChecking=no ${SSH_USER}@${ANSIBLE_HOST} <<'EOF'
              set -e
              export AZURE_CLIENT_ID=$AZURE_CLIENT_ID
              export AZURE_SECRET=$AZURE_SECRET
              export AZURE_TENANT=$AZURE_TENANT
              export AZURE_SUBSCRIPTION_ID=$AZURE_SUBSCRIPTION_ID

              rm -rf ${WORKDIR}
              git clone ${REPO_URL} ${WORKDIR}

              source ${VENV_PATH}/bin/activate
              cd ${WORKDIR}/playbooks
              ansible-playbook ${playbook} -e "@../config/config-dev.yaml"

              rm -rf ${WORKDIR}
              EOF
            """
          }
        }
      }
    }

    stage('🌐 ดึง IP (เฉพาะเมื่อสร้าง)') {
      when {
        expression { return env.DESTROY_MODE.toLowerCase() == "false" }
      }
      steps {
        withCredentials([
          sshUserPrivateKey(credentialsId: 'ssh-ansible-agent', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER'),
          string(credentialsId: 'AZURE_CLIENT_ID',       variable: 'AZURE_CLIENT_ID'),
          string(credentialsId: 'AZURE_SECRET',          variable: 'AZURE_SECRET'),
          string(credentialsId: 'AZURE_TENANT',          variable: 'AZURE_TENANT'),
          string(credentialsId: 'AZURE_SUBSCRIPTION_ID', variable: 'AZURE_SUBSCRIPTION_ID')
        ]) {
          sh """
            ssh -i $SSH_KEY -o StrictHostKeyChecking=no ${SSH_USER}@${ANSIBLE_HOST} <<'EOF'
            set -e
            export AZURE_CLIENT_ID=$AZURE_CLIENT_ID
            export AZURE_SECRET=$AZURE_SECRET
            export AZURE_TENANT=$AZURE_TENANT
            export AZURE_SUBSCRIPTION_ID=$AZURE_SUBSCRIPTION_ID

            rm -rf ${WORKDIR}
            git clone ${REPO_URL} ${WORKDIR}

            source ${VENV_PATH}/bin/activate
            cd ${WORKDIR}/playbooks
            ansible-playbook get-vm-ip.yaml -e "@../config/config-dev.yaml"

            echo "✅ IP ที่ได้:"
            cat vm_ip.txt

            rm -rf ${WORKDIR}
            EOF
          """
        }
      }
    }
  }
}
