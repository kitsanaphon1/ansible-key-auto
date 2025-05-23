pipeline {
  agent any // ✅ ใช้ Jenkins main node หรือ agent ใด ๆ ก็ได้

  environment {
    GIT_BRANCH       = "dev"                                           // 📌 ชื่อ branch ที่จะ clone (ใช้ในอนาคตถ้ต้องการบังคับ branch)
    ANSIBLE_HOST     = "4.145.84.26"                                   // 📍 IP ของ Ansible VM ที่ Jenkins จะ SSH เข้าไป
    DESTROY_MODE     = "false"                                         // 🔁 กำหนดว่าเป็นโหมดลบ VM หรือสร้าง VM
    VENV_PATH        = "/home/boho/ansible-env"                        // 🐍 Python venv ที่ติดตั้ง Ansible ไว้ใน Ansible VM
    WORKDIR          = "/tmp/ansible-key-auto-run"                     // 📁 โฟลเดอร์ชั่วคราวที่ใช้ clone repo
    REPO_URL         = "https://github.com/kitsanaphon1/ansible-key-auto.git" // 🔗 Git repo ที่เก็บ playbook
  }

  stages {

    stage('📥 Checkout Jenkinsfile') {
      steps {
        checkout scm // 🔄 ดึง Jenkinsfile จาก repo (เพื่อให้รู้ pipeline ที่ต้องรัน)
      }
    }

    stage('☁️ Provision หรือ Destroy VM (ผ่าน clone repo)') {
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
              echo "🚀 SSH เข้า Ansible VM และรัน playbook แบบ clone ชั่วคราว"
              ssh -i $SSH_KEY -o StrictHostKeyChecking=no ${SSH_USER}@${ANSIBLE_HOST} <<EOF
              set -e
              export AZURE_CLIENT_ID=$AZURE_CLIENT_ID
              export AZURE_SECRET=$AZURE_SECRET
              export AZURE_TENANT=$AZURE_TENANT
              export AZURE_SUBSCRIPTION_ID=$AZURE_SUBSCRIPTION_ID

              # 💥 ล้าง repo เก่า (ถ้ามี)
              rm -rf ${WORKDIR}

              # 📥 clone repo ใหม่
              git clone ${REPO_URL} ${WORKDIR}

              # 🐍 เปิด virtual environment
              source ${VENV_PATH}/bin/activate

              # 🛠️ รัน playbook
              cd ${WORKDIR}/playbooks
              ansible-playbook ${playbook} -e "@../config/config-dev.yaml"

              # 🧹 ล้าง repo ชั่วคราวออก
              rm -rf ${WORKDIR}
              EOF
            """
          }
        }
      }
    }

    stage('🌐 ดึง IP (เฉพาะเมื่อสร้าง)') {
      when {
        expression { return env.DESTROY_MODE.toLowerCase() == "false" } // ❌ ข้ามขั้นตอนนี้ถ้าเป็นโหมดลบ
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
            echo "🌐 SSH ไปดึง IP หลังสร้าง VM"
            ssh -i $SSH_KEY -o StrictHostKeyChecking=no ${SSH_USER}@${ANSIBLE_HOST} <<EOF
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
