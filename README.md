# jenkins-docker-pipeline
This repository provides a step-by-step guide to setting up a Jenkins Master-Slave architecture on AWS EC2 instances. This configuration allows you to offload build workloads to a dedicated agent node equipped with Docker.


### Step 1: Infrastructure & Security Setup

Launch two **Ubuntu** instances and configure their Security Groups as follows:

* **Instance 1 (Jenkins-Master):** Open port `8080` (Jenkins) and `22` (SSH).
* **Instance 2 (Jenkins-Slave):** Open port `22` (SSH).

---

### Step 2: Install Jenkins on Master

Connect to your **Master** instance via SSH and run:

**Update and Install Java:**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install openjdk-17-jdk -y

```

**Add Jenkins Repository:**

```bash
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

```

**Install and Start Jenkins:**

```bash
sudo apt update
sudo apt install jenkins -y
sudo systemctl start jenkins
sudo systemctl enable jenkins

```

Access the Jenkins at `http://<Master-Public-IP>:8080`. Retrieve the initial admin password with:
`sudo cat /var/lib/jenkins/secrets/initialAdminPassword`

---

### Step 3: Configure the Slave Node

Connect to your **Slave** instance via SSH to prepare the environment.

**Install Java & Docker:**

```bash
sudo apt update
sudo apt install openjdk-17-jdk docker.io -y
sudo systemctl start docker
sudo systemctl enable docker

```

**Add Jenkins User:**

```bash
sudo useradd -m -d /home/jenkins -s /bin/bash jenkins
sudo usermod -aG docker jenkins # Grant permission to use Docker

```

**Set up SSH Authentication:**

1. Switch to the jenkins user: `sudo su - jenkins`
2. Create folder: `mkdir .ssh && chmod 700 .ssh`
3. On your **Local Machine**, generate the public key from your `.pem` key:
`ssh-keygen -y -f <your-aws-key>.pem`
4. Copy that output and paste it into `/home/jenkins/.ssh/authorized_keys` on the Slave.

---

### Step 4: Link Slave to Master

Jenkins Dashboard in the browser:

1. **Add Credentials:**
* Go to **Manage Jenkins > Credentials > System > Global credentials > Add Credentials**.
* **Kind:** SSH Username with private key.
* **ID:** `slave-ssh-key` | **Username:** `jenkins`
* **Private Key:** Paste the contents of your local `.pem` file.


2. **Create the Node:**
* Go to **Manage Jenkins > Nodes > New Node**.
* **Name:** `Docker-Slave` | **Type:** Permanent Agent.
* **Remote root directory:** `/home/jenkins`
* **Labels:** `docker-agent`
* **Launch method:** Launch agents via SSH.
* **Host:** `<Slave-Private-IP>`
* **Credentials:** Select `slave-ssh-key`.
* **Host Key Verification Strategy:** "Non verifying Verification Strategy".



---

### Step 5: Verify via Pipeline

Create a "New Item" (Pipeline) and use this script to verify the distributed setup:

```groovy
pipeline {
    agent { label 'docker-agent' } // Forces the job to run on the Slave node
    stages {
        stage('Test Docker') {
            steps {
                sh 'docker --version'
                sh 'docker run hello-world'
            }
        }
    }
}

```

## Troubleshooting

* **Connection Denied:** Ensure the Master can reach the Slave's Private IP on port 22.
* **Java Mismatch:** Ensure both Master and Slave are running `openjdk-17-jdk`.
