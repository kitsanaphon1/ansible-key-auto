pipeline {
  agent any // ✅ ใช้ Jenkins main node หรือ agent ใด ๆ ก็ได้

  environment {
    GIT_BRANCH       = "dev" // 🛠️ ระบุชื่อ branch ของ Git (ยังไม่ได้ใช้ใน pipeline นี้ แต่สามารถนำไปใช้เพิ่มได้ภายหลัง)
    ANSIBLE_HOST     = "4.145.84.26" // 🌐 IP ของ Ansible VM ที่ Jenkins จะ SSH เข้าไปเพื่อรัน playbook
    DESTROY_MODE     = "false" // 🔁 ถ้า true จะรัน playbook ลบ VM แทนที่จะสร้าง   false จะสรา้ง vm 
    VENV_PATH        = "/home/boho/ansible-env" // 🐍 Python Virtual Environment ที่ติดตั้ง Ansible ไว้ใน Ansible VM
    WORKDIR          = "/tmp/ansible-key-auto-run" // 📁 โฟลเดอร์ชั่วคราวบน Ansible VM ที่จะใช้ clone repo
    REPO_URL         = "https://github.com/kitsanaphon1/ansible-key-auto.git" // 🔗 Git repo ที่เก็บ playbooks และ config ทั้งหมด
  }

  stages {

    stage('📥 Checkout Jenkinsfile') {
      steps {
        checkout scm // 🔄 ดึง source code จาก repo ปัจจุบัน เพื่อโหลด Jenkinsfile นี้
      }
    }

    stage('☁️ Provision หรือ Destroy VM') {
      steps {
        withCredentials([
          // 🔐 SSH key สำหรับเชื่อมต่อ Ansible VM
          sshUserPrivateKey(credentialsId: 'ssh-ansible-agent', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER'),
          // ☁️ Credentials ที่ใช้เชื่อมต่อ Azure API
          string(credentialsId: 'AZURE_CLIENT_ID',       variable: 'AZURE_CLIENT_ID'),
          string(credentialsId: 'AZURE_SECRET',          variable: 'AZURE_SECRET'),
          string(credentialsId: 'AZURE_TENANT',          variable: 'AZURE_TENANT'),
          string(credentialsId: 'AZURE_SUBSCRIPTION_ID', variable: 'AZURE_SUBSCRIPTION_ID')
        ]) {
          script {
            // 🧠 เลือก playbook ตาม DESTROY_MODE
            def playbook = (DESTROY_MODE == "true") ? "destroy-linux-vm.yaml" : "create-linux-vm.yaml"

            sh """
ssh -i $SSH_KEY -o StrictHostKeyChecking=no ${SSH_USER}@${ANSIBLE_HOST} <<'EOF'
set -e
export AZURE_CLIENT_ID=$AZURE_CLIENT_ID
export AZURE_SECRET=$AZURE_SECRET
export AZURE_TENANT=$AZURE_TENANT
export AZURE_SUBSCRIPTION_ID=$AZURE_SUBSCRIPTION_ID

# 🧹 ลบโฟลเดอร์ clone เก่า
rm -rf ${WORKDIR}

# 📥 clone Git repo ลงเครื่อง Ansible
git clone ${REPO_URL} ${WORKDIR}

# 🐍 เข้า Python venv และรัน playbook ที่สร้าง/ลบ VM
source ${VENV_PATH}/bin/activate
cd ${WORKDIR}/playbooks
ansible-playbook ${playbook} -e "@../config/config-dev.yaml"

# 🧼 ล้าง repo ที่ clone ทิ้งเมื่อเสร็จ
rm -rf ${WORKDIR}
EOF
"""
          }
        }
      }
    }

    stage('🌐 ดึง IP (เฉพาะเมื่อสร้าง)') {
      when {
        expression { return env.DESTROY_MODE.toLowerCase() == "false" } // 🚫 ถ้า DESTROY_MODE เป็น true ให้ข้าม stage นี้
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

# 🧾 รัน playbook ที่ใช้ดึง Public IP และบันทึกลงไฟล์ vm_ip.txt
ansible-playbook get-vm-ip.yaml -e "@../config/config-dev.yaml"

# 👀 แสดงค่า IP ที่ดึงได้ (ถ้ามี)
echo "✅ IP ที่ได้:"
cat vm_ip.txt || echo '⚠️ ไม่พบไฟล์ vm_ip.txt'

rm -rf ${WORKDIR}
EOF
"""
        }
      }
    }
  }
}
