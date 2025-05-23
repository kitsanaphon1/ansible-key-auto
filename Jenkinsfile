pipeline {
  agent { label 'ansible-agent' } // ✅ กำหนดให้ Pipeline รันบน Jenkins agent ที่มี Python + Ansible ติดตั้งแล้ว

  environment {
    GIT_BRANCH       = "dev"                        // 🛠️ สำหรับใช้กรณีต้องการเช็ค branch หรือ trigger เฉพาะ branch
    ANSIBLE_HOST     = "4.145.84.26"                // 🌍 IP ของเครื่อง Ansible VM ที่ Jenkins จะ SSH เข้าไป
    DESTROY_MODE     = "false"                      // 🔁 เปลี่ยนเป็น "true" ถ้าต้องการลบ VM แทนการสร้าง
    ANSIBLE_ENV_PATH = "/home/boho/ansible-env"     // 🐍 path ไปยัง Python venv ที่ติดตั้ง Ansible
    PROJECT_DIR      = "/home/boho/ANSIBLE-KEY-AUTO"// 📁 โฟลเดอร์ที่เก็บ playbooks + config บน Ansible VM
  }

  stages {
    stage('📥 Checkout Source Code') {
      steps {
        checkout scm // 🔄 ดึง source code จาก Git repo ที่ผูกไว้กับ Jenkins job นี้
      }
    }

    stage('🧹 Clean Workspace') {
      steps {
        cleanWs() // 🧼 ล้าง workspace เดิม ป้องกันไฟล์ซ้ำหรือปัญหาการรันซ้ำ
      }
    }

    stage('☁️ Provision หรือ Destroy VM') {
      steps {
        withCredentials([
          // 🔐 ดึง SSH credential ที่สร้างไว้ใน Jenkins UI
          sshUserPrivateKey(credentialsId: 'ssh-ansible-agent', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER'),

          // ☁️ ดึง Azure credentials จาก Jenkins credential store มาเป็น environment variables
          string(credentialsId: 'AZURE_CLIENT_ID',       variable: 'AZURE_CLIENT_ID'),
          string(credentialsId: 'AZURE_SECRET',          variable: 'AZURE_SECRET'),
          string(credentialsId: 'AZURE_TENANT',          variable: 'AZURE_TENANT'),
          string(credentialsId: 'AZURE_SUBSCRIPTION_ID', variable: 'AZURE_SUBSCRIPTION_ID')
        ]) {
          script {
            // 🧠 เลือกว่าจะใช้ playbook ไหน (สร้าง VM หรือ ลบ VM) ตาม DESTROY_MODE
            def playbook = (DESTROY_MODE == "true") ? "destroy-linux-vm.yaml" : "create-linux-vm.yaml"

            // 🚀 SSH เข้าเครื่อง Ansible แล้ว activate venv และรัน playbook
            sh """
              ssh -i $SSH_KEY -o StrictHostKeyChecking=no ${SSH_USER}@${ANSIBLE_HOST} <<EOF
              export AZURE_CLIENT_ID=$AZURE_CLIENT_ID
              export AZURE_SECRET=$AZURE_SECRET
              export AZURE_TENANT=$AZURE_TENANT
              export AZURE_SUBSCRIPTION_ID=$AZURE_SUBSCRIPTION_ID

              source ${ANSIBLE_ENV_PATH}/bin/activate           # 🔧 เปิด Python venv ที่มี Ansible ติดตั้งไว้
              cd ${PROJECT_DIR}/playbooks                       # 📁 เข้าโฟลเดอร์ที่เก็บ playbooks
              ansible-playbook ${playbook} -e "@../config/config-dev.yaml"  # 🛠️ รัน playbook พร้อม config
              EOF
            """
          }
        }
      }
    }

    stage('🌐 ดึง IP VM (ถ้าไม่ได้ destroy)') {
      when {
        expression { return env.DESTROY_MODE.toLowerCase() == "false" } // 🚫 ข้ามขั้นตอนนี้ถ้าเป็นโหมดลบ
      }
      steps {
        withCredentials([
          sshUserPrivateKey(credentialsId: 'ssh-ansible-agent', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER'),
          string(credentialsId: 'AZURE_CLIENT_ID',       variable: 'AZURE_CLIENT_ID'),
          string(credentialsId: 'AZURE_SECRET',          variable: 'AZURE_SECRET'),
          string(credentialsId: 'AZURE_TENANT',          variable: 'AZURE_TENANT'),
          string(credentialsId: 'AZURE_SUBSCRIPTION_ID', variable: 'AZURE_SUBSCRIPTION_ID')
        ]) {
          // 🌐 รัน get-vm-ip.yaml บนเครื่อง Ansible เพื่อดึง Public IP ของ VM ที่เพิ่งสร้าง
          sh """
            ssh -i $SSH_KEY -o StrictHostKeyChecking=no ${SSH_USER}@${ANSIBLE_HOST} <<EOF
            export AZURE_CLIENT_ID=$AZURE_CLIENT_ID
            export AZURE_SECRET=$AZURE_SECRET
            export AZURE_TENANT=$AZURE_TENANT
            export AZURE_SUBSCRIPTION_ID=$AZURE_SUBSCRIPTION_ID

            source ${ANSIBLE_ENV_PATH}/bin/activate
            cd ${PROJECT_DIR}/playbooks
            ansible-playbook get-vm-ip.yaml -e "@../config/config-dev.yaml"
            echo "✅ Public IP:"
            cat vm_ip.txt                                           # 🖨️ แสดง IP ที่ได้ไว้ใน Console
            EOF
          """
        }
      }
    }
  }
}
