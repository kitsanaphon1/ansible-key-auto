pipeline {
  agent any // ✅ ไม่ใช้ Jenkins agent แยก

  environment {
    GIT_BRANCH       = "dev"
    ANSIBLE_HOST     = "4.145.84.26"                  // IP ของ Ansible VM ที่ Jenkins จะ SSH เข้าไป
    DESTROY_MODE     = "false"                        // true = ลบ, false = สร้าง
    ANSIBLE_ENV_PATH = "/home/boho/ansible-env"       // venv ที่มี ansible ติดตั้งใน Ansible VM
    PROJECT_DIR      = "/home/boho/ANSIBLE-KEY-AUTO"  // โฟลเดอร์ repo บน Ansible VM
  }

  stages {
    stage('📥 Checkout Source Code') {
      steps {
        checkout scm // ดึง Jenkinsfile จาก Git
      }
    }

    stage('🧹 Clean Workspace') {
      steps {
        cleanWs()
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
              echo "🚀 SSH ไป Ansible VM และรัน ${playbook}"
              ssh -i $SSH_KEY -o StrictHostKeyChecking=no ${SSH_USER}@${ANSIBLE_HOST} <<EOF
              export AZURE_CLIENT_ID=$AZURE_CLIENT_ID
              export AZURE_SECRET=$AZURE_SECRET
              export AZURE_TENANT=$AZURE_TENANT
              export AZURE_SUBSCRIPTION_ID=$AZURE_SUBSCRIPTION_ID

              source ${ANSIBLE_ENV_PATH}/bin/activate
              cd ${PROJECT_DIR}/playbooks
              ansible-playbook ${playbook} -e "@../config/config-dev.yaml"
              EOF
            """
          }
        }
      }
    }

    stage('🌐 ดึง IP VM') {
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
            echo "🌐 SSH ไป Ansible VM เพื่อดึง Public IP"
            ssh -i $SSH_KEY -o StrictHostKeyChecking=no ${SSH_USER}@${ANSIBLE_HOST} <<EOF
            export AZURE_CLIENT_ID=$AZURE_CLIENT_ID
            export AZURE_SECRET=$AZURE_SECRET
            export AZURE_TENANT=$AZURE_TENANT
            export AZURE_SUBSCRIPTION_ID=$AZURE_SUBSCRIPTION_ID

            source ${ANSIBLE_ENV_PATH}/bin/activate
            cd ${PROJECT_DIR}/playbooks
            ansible-playbook get-vm-ip.yaml -e "@../config/config-dev.yaml"
            echo "✅ Public IP:"
            cat vm_ip.txt
            EOF
          """
        }
      }
    }
  }
}
