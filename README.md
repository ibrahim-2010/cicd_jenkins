# End-to-End CI/CD Pipeline: Jenkins → Docker → Amazon EKS with Monitoring

![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=for-the-badge&logo=jenkins&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=for-the-badge&logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white)

A fully automated CI/CD pipeline that takes application code from GitHub, runs unit tests, builds a Docker image, pushes it to DockerHub, and deploys it to an Amazon EKS Kubernetes cluster — with Prometheus and Grafana monitoring and custom alerting.

## 📋 Table of Contents

- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Step 1: Launch EC2 Instance for Jenkins](#step-1-launch-ec2-instance-for-jenkins)
- [Step 2: Install Jenkins](#step-2-install-jenkins)
- [Step 3: Install Docker](#step-3-install-docker)
- [Step 4: Install AWS CLI v2](#step-4-install-aws-cli-v2)
- [Step 5: Install kubectl](#step-5-install-kubectl)
- [Step 6: Install eksctl](#step-6-install-eksctl)
- [Step 7: Install Helm](#step-7-install-helm)
- [Step 8: Configure Jenkins](#step-8-configure-jenkins)
- [Step 9: Store Credentials in Jenkins](#step-9-store-credentials-in-jenkins)
- [Step 10: Create the EKS Cluster](#step-10-create-the-eks-cluster)
- [Step 11: Create Kubernetes Manifests](#step-11-create-kubernetes-manifests)
- [Step 12: Create the Jenkinsfile](#step-12-create-the-jenkinsfile)
- [Step 13: Create the Jenkins Pipeline Job](#step-13-create-the-jenkins-pipeline-job)
- [Step 14: Run the Pipeline](#step-14-run-the-pipeline)
- [Step 15: Install Prometheus and Grafana](#step-15-install-prometheus-and-grafana)
- [Step 16: Set Up Custom Alerts](#step-16-set-up-custom-alerts)
- [Step 17: Test the Alerts](#step-17-test-the-alerts)
- [Cleanup](#cleanup)
- [Troubleshooting](#troubleshooting)
- [Cost Estimate](#cost-estimate)

---

## Architecture

```
┌──────────┐     ┌──────────┐     ┌───────────┐     ┌───────────┐     ┌───────────┐     ┌───────────┐
│  GitHub   │────▶│  Jenkins  │────▶│   Maven    │────▶│  Docker   │────▶│ DockerHub │────▶│  Amazon   │
│   Repo    │     │  Server   │     │   Tests    │     │  Build    │     │   Push    │     │   EKS     │
└──────────┘     └──────────┘     └───────────┘     └───────────┘     └───────────┘     └───────────┘
                                                                                              │
                                                                                    ┌─────────┴─────────┐
                                                                                    │   Monitoring       │
                                                                                    │  Prometheus +      │
                                                                                    │  Grafana + Alerts  │
                                                                                    └───────────────────┘
```

**Pipeline Stages:**

| Stage | Action | Duration |
|-------|--------|----------|
| Cleanup | Wipe workspace for clean build | ~600ms |
| Checkout Repo | Clone main branch from GitHub | ~1s |
| Unit Testing | Run Maven/JUnit tests | ~7s |
| DockerBuild | Build image with BUILD_NUMBER tag | ~2s |
| DockerPush | Push tagged image to DockerHub | ~3s |
| Deploy to EKS | Apply K8s manifests to cluster | ~4s |

**Total pipeline time: ~25 seconds**

---

## Prerequisites

Before starting, make sure you have:

- An **AWS account** with admin access
- A **DockerHub account** (free tier works)
- A **GitHub account**
- Basic knowledge of Linux command line
- An **SSH client** (Terminal on Mac/Linux, Git Bash or PuTTY on Windows)
- An **AWS key pair** (.pem file) for SSH access to EC2

---

## Project Structure

```
cicd_jenkins/
├── Dockerfile              # Nginx-based container image
├── index.html              # Application homepage
├── Jenkinsfile             # CI/CD pipeline definition
├── pom.xml                 # Maven build configuration
├── README.md               # This file
├── k8s/
│   └── deployment.yaml     # Kubernetes Deployment + Service
└── src/
    ├── main/java/com/brymo/App.java
    └── test/java/com/brymo/AppTest.java
```

---

## Step 1: Launch EC2 Instance for Jenkins

1. Log in to the **AWS Console** → **EC2** → **Launch Instance**

2. Configure the instance:
   - **Name:** `jenkins-server`
   - **AMI:** Ubuntu Server 24.04 LTS
   - **Instance type:** `t3.small` (minimum for Jenkins + Docker)
   - **Key pair:** Select or create one
   - **Security Group:** Create with these inbound rules:

     | Port | Protocol | Source | Purpose |
     |------|----------|--------|---------|
     | 22 | TCP | Your IP | SSH access |
     | 8080 | TCP | 0.0.0.0/0 | Jenkins UI |
     | 80 | TCP | 0.0.0.0/0 | HTTP (optional) |

3. Launch the instance and SSH into it:

```bash
ssh -i your-key.pem ubuntu@<your-ec2-public-ip>
```

4. Switch to root:

```bash
sudo su -
```

---

## Step 2: Install Jenkins

```bash
# Update system packages
apt update && apt upgrade -y

# Install Java (required for Jenkins)
apt install -y fontconfig openjdk-17-jre

# Verify Java
java -version

# Add Jenkins repository
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install Jenkins
apt update
apt install -y jenkins

# Start Jenkins
systemctl start jenkins
systemctl enable jenkins

# Get the initial admin password
cat /var/lib/jenkins/secrets/initialAdminPassword
```

**Access Jenkins:**
1. Open `http://<your-ec2-public-ip>:8080` in your browser
2. Paste the initial admin password
3. Install **suggested plugins**
4. Create your admin user

---

## Step 3: Install Docker

```bash
# Install Docker
apt install -y docker.io

# Start Docker
systemctl start docker
systemctl enable docker

# Add Jenkins user to Docker group (so Jenkins can run Docker commands)
usermod -aG docker jenkins

# Restart Jenkins to pick up the group change
systemctl restart jenkins

# Verify Docker
docker --version
```

---

## Step 4: Install AWS CLI v2

> **Note:** Do NOT use `apt install awscli` — the package is outdated or unavailable on many Ubuntu versions. Use the official AWS installer instead.

```bash
# Download and install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
apt install -y unzip
unzip awscliv2.zip
./aws/install

# Verify
aws --version

# Clean up
rm -rf awscliv2.zip aws/
```

**Configure AWS credentials:**

```bash
aws configure
```

Enter:
- **Access Key ID:** Your AWS access key
- **Secret Access Key:** Your AWS secret key
- **Default region:** `us-east-2` (or your preferred region)
- **Output format:** `json`

Verify:

```bash
aws sts get-caller-identity
```

---

## Step 5: Install kubectl

```bash
# Download kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Install
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify
kubectl version --client

# Clean up
rm kubectl
```

---

## Step 6: Install eksctl

```bash
# Download and install eksctl
curl --silent --location "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Verify
eksctl version
```

---

## Step 7: Install Helm

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version
```

---

## Step 8: Configure Jenkins

### Install Required Plugins

1. Go to **Jenkins** → **Manage Jenkins** → **Plugins** → **Available plugins**
2. Search for and install:
   - **Maven Integration**
   - **Pipeline**
   - **Git**
   - **Docker Pipeline**
   - **Workspace Cleanup**

### Configure JDK

1. Go to **Manage Jenkins** → **Tools**
2. Under **JDK installations**, click **Add JDK**
   - Name: `openjdk`
   - Uncheck "Install automatically"
   - JAVA_HOME: `/usr/lib/jvm/java-17-openjdk-amd64`

### Configure Maven

1. Under **Maven installations**, click **Add Maven**
   - Name: `mvn`
   - Check "Install automatically"
   - Select the latest version

Click **Save**.

---

## Step 9: Store Credentials in Jenkins

Jenkins needs DockerHub credentials to push images and AWS credentials to deploy to EKS. **Never hardcode credentials in your Jenkinsfile.**

### DockerHub Credentials

1. Go to **Jenkins** → **Manage Jenkins** → **Credentials** → **System** → **Global credentials** → **Add Credentials**
2. Fill in:
   - **Kind:** Username with password
   - **Username:** Your DockerHub username (e.g., `ibj2010`)
   - **Password:** Your DockerHub password or access token
   - **ID:** `jenkins_credentials`
3. Click **Create**

### AWS Credentials

You need to create **two** Secret Text entries:

**AWS Access Key:**
1. **Add Credentials** → **Kind:** Secret text
2. **Secret:** Paste your AWS Access Key ID
3. **ID:** `aws-access-key`
4. Click **Create**

**AWS Secret Key:**
1. **Add Credentials** → **Kind:** Secret text
2. **Secret:** Paste your AWS Secret Access Key
3. **ID:** `aws-secret-key`
4. Click **Create**

You should now have **3 credentials** listed: `jenkins_credentials`, `aws-access-key`, `aws-secret-key`.

---

## Step 10: Create the EKS Cluster

> **⏱ This takes 15-20 minutes.** Start it and move on to the next steps while it creates.

```bash
eksctl create cluster \
  --name ibra \
  --region us-east-2 \
  --node-type t3.small \
  --nodes 2
```

> **Why t3.small?** In `us-east-2`, `t3.small` typically has larger available capacity than `t3.medium`, reducing the chance of cluster creation failures due to insufficient capacity.

Once complete, verify:

```bash
# Update kubeconfig
aws eks update-kubeconfig --name ibra --region us-east-2

# Check nodes are ready
kubectl get nodes
```

Expected output:

```
NAME                                           STATUS   ROLES    AGE   VERSION
ip-192-168-5-238.us-east-2.compute.internal    Ready    <none>   3m    v1.34.x
ip-192-168-88-152.us-east-2.compute.internal   Ready    <none>   3m    v1.34.x
```

---

## Step 11: Create Kubernetes Manifests

Create the `k8s/` directory and deployment manifest in your project:

```bash
mkdir -p k8s
```

Create `k8s/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ibra-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ibra-app
  template:
    metadata:
      labels:
        app: ibra-app
    spec:
      containers:
        - name: ibra-app
          image: IMAGE_PLACEHOLDER
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: ibra-app-service
spec:
  type: LoadBalancer
  selector:
    app: ibra-app
  ports:
    - port: 80
      targetPort: 80
```

**Key points:**
- `IMAGE_PLACEHOLDER` gets replaced by the Jenkinsfile at deploy time with the actual image and build number
- `replicas: 2` runs two pods for high availability
- `type: LoadBalancer` provisions an AWS ELB to expose the app publicly

---

## Step 12: Create the Jenkinsfile

Create `Jenkinsfile` in the root of your project:

```groovy
pipeline {
    agent any
    tools {
        maven 'mvn'
        jdk   'openjdk'
    }
    environment {
        DOCKER_CREDS          = credentials('jenkins_credentials')
        AWS_ACCESS_KEY_ID     = credentials('aws-access-key')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
        AWS_DEFAULT_REGION    = 'us-east-2'
    }
    stages {
        stage('Cleanup') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout Repo') {
            steps {
                git branch: 'main', url: 'https://github.com/ibrahim-2010/cicd_jenkins.git'
            }
        }
        stage('Unit Testing') {
            steps {
                sh 'mvn test'
            }
        }
        stage('DockerBuild') {
            steps {
                sh "docker build -t ibj2010/nginx:${BUILD_NUMBER} ."
            }
        }
        stage('DockerPush') {
            steps {
                sh '''
                    echo $DOCKER_CREDS_PSW | docker login -u $DOCKER_CREDS_USR --password-stdin
                '''
                sh "docker push ibj2010/nginx:${BUILD_NUMBER}"
            }
        }
        stage('Deploy to EKS') {
            steps {
                script {
                    // Update kubeconfig to connect to EKS cluster
                    sh 'aws eks update-kubeconfig --name ibra --region us-east-2'
                    // Replace placeholder with actual image tag
                    sh "sed -i 's|IMAGE_PLACEHOLDER|ibj2010/nginx:${BUILD_NUMBER}|g' k8s/deployment.yaml"
                    // Apply the Kubernetes manifests
                    sh 'kubectl apply -f k8s/deployment.yaml'
                    // Verify rollout completes successfully
                    sh 'kubectl rollout status deployment/ibra-app --timeout=120s'
                }
            }
        }
    }
}
```

> **Important:** Replace `ibj2010/nginx` with your own DockerHub `username/image-name` and update the GitHub URL to your own repository.

**Commit and push everything:**

```bash
git add Jenkinsfile k8s/deployment.yaml
git commit -m "Add Jenkinsfile with EKS deploy stage and k8s manifests"
git push origin main
```

---

## Step 13: Create the Jenkins Pipeline Job

1. In Jenkins, click **New Item**
2. Enter a name (e.g., `ibra`)
3. Select **Pipeline** → **OK**
4. Under **Pipeline**, set:
   - **Definition:** Pipeline script from SCM
   - **SCM:** Git
   - **Repository URL:** `https://github.com/ibrahim-2010/cicd_jenkins.git`
   - **Branch:** `*/main`
   - **Script Path:** `Jenkinsfile`
5. Click **Save**

---

## Step 14: Run the Pipeline

1. Click **Build Now**
2. Watch the **Stage View** — all 6 stages should turn green
3. Verify the deployment:

```bash
# Check deployment
kubectl get deployments

# Check pods
kubectl get pods

# Get the external URL
kubectl get svc ibra-app-service
```

4. Copy the `EXTERNAL-IP` from the service and open it in your browser — you should see your application!

---

## Step 15: Install Prometheus and Grafana

Deploy the full monitoring stack using Helm:

```bash
# Add the Prometheus community Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create a monitoring namespace
kubectl create namespace monitoring

# Install kube-prometheus-stack (includes Prometheus + Grafana + Alertmanager)
helm install monitoring prometheus-community/kube-prometheus-stack --namespace monitoring

# Verify all pods are running
kubectl get pods -n monitoring
```

Expected pods (wait until all show `Running`):

```
alertmanager-monitoring-kube-prometheus-alertmanager-0    2/2     Running
monitoring-grafana-xxxxxxxxxx-xxxxx                       3/3     Running
monitoring-kube-prometheus-operator-xxxxxxxxxx-xxxxx      1/1     Running
monitoring-kube-state-metrics-xxxxxxxxxx-xxxxx            1/1     Running
monitoring-prometheus-node-exporter-xxxxx                 1/1     Running
monitoring-prometheus-node-exporter-xxxxx                 1/1     Running
prometheus-monitoring-kube-prometheus-prometheus-0        2/2     Running
```

### Access Grafana

```bash
# Expose Grafana via LoadBalancer
kubectl patch svc monitoring-grafana -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'

# Get the external URL
kubectl get svc monitoring-grafana -n monitoring
```

**Get the Grafana password:**

```bash
kubectl get secret monitoring-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode; echo
```

**Login:**
- **URL:** `http://<EXTERNAL-IP>` from the service
- **Username:** `admin`
- **Password:** Output from the command above

Once logged in, go to **Dashboards → Browse** to explore pre-built Kubernetes dashboards.

---

## Step 16: Set Up Custom Alerts

Create a PrometheusRule with custom alerts for your application:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ibra-app-alerts
  namespace: monitoring
  labels:
    release: monitoring
spec:
  groups:
    - name: ibra-app.rules
      rules:
        - alert: PodDown
          expr: kube_deployment_status_replicas_available{deployment="ibra-app"} < 2
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "ibra-app pod is down"
            description: "Available replicas for ibra-app is below 2. Current value: {{ $value }}"

        - alert: HighCPUUsage
          expr: rate(container_cpu_usage_seconds_total{namespace="default", pod=~"ibra-app.*"}[5m]) > 0.5
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: "High CPU usage on ibra-app"
            description: "CPU usage for ibra-app pod is above 50%. Current value: {{ $value }}"

        - alert: PodCrashLooping
          expr: rate(kube_pod_container_status_restarts_total{namespace="default", pod=~"ibra-app.*"}[15m]) > 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "ibra-app pod is crash looping"
            description: "Pod {{ $labels.pod }} is restarting frequently"
EOF
```

Verify the rules were created:

```bash
kubectl get prometheusrules -n monitoring
```

You should see `ibra-app-alerts` in the list.

**View alerts in Grafana:**
Go to **Alerting → Alert Rules** → scroll to find `ibra-app.rules` under **Data source-managed**.

---

## Step 17: Test the Alerts

### Trigger the PodDown Alert

Scale the deployment down to 1 replica (below the threshold of 2):

```bash
kubectl scale deployment ibra-app --replicas=1
```

Wait ~1 minute, then check Grafana → **Alerting → Alert Rules**. The **PodDown** alert should transition:

```
Normal → Pending → Firing
```

### Restore the Deployment

```bash
kubectl scale deployment ibra-app --replicas=2
```

The alert should return to **Normal** after about a minute.

---

## Cleanup

> ⚠️ **Important:** EKS clusters cost money. Always clean up when you're done.

```bash
# Delete the EKS cluster (removes nodes, VPC, load balancers)
eksctl delete cluster --name ibra --region us-east-2

# Terminate the Jenkins EC2 instance
# Do this from the AWS Console → EC2 → select instance → Instance State → Terminate
```

---

## Troubleshooting

### Common Issues and Fixes

| Problem | Cause | Solution |
|---------|-------|---------|
| `apt install awscli` fails with "no installation candidate" | Package not in Ubuntu repos | Use the official AWS CLI v2 bundled installer (see [Step 4](#step-4-install-aws-cli-v2)) |
| `eksctl: command not found` | eksctl not installed | Install via tarball (see [Step 6](#step-6-install-eksctl)) |
| `Error: failed to create cluster` — exceeded max wait time | Insufficient capacity for instance type in AZ | Use `t3.small` instead of `t3.medium`, or specify `--zones` |
| Jenkins Deploy stage: `Unable to locate credentials` | AWS CLI configured for root, not jenkins user | Store AWS credentials in Jenkins Credentials Manager as Secret text (see [Step 9](#step-9-store-credentials-in-jenkins)) |
| `sed: can't read k8s/deployment.yaml: No such file or directory` | File not pushed to GitHub | Run `git add k8s/deployment.yaml && git commit && git push` |
| Docker image tag mismatch (push `:latest` but deploy uses `:BUILD_NUMBER`) | Inconsistent tagging across stages | Use `${BUILD_NUMBER}` in both DockerBuild and DockerPush stages |
| `ERROR: aws-access-key` in Jenkins | Credential ID doesn't exist in Jenkins | Create the credential in Jenkins with the exact ID: `aws-access-key` |
| Grafana login fails with `prom-operator` | Password was changed or differs from default | Retrieve actual password: `kubectl get secret monitoring-grafana -n monitoring -o jsonpath="{.data.admin-password}" \| base64 --decode` |
| `KubeControllerManagerDown` / `KubeSchedulerDown` firing | Normal on EKS — AWS manages control plane | These can be safely ignored on managed EKS |

---

## Cost Estimate

| Resource | Cost (approx.) |
|----------|---------------|
| EC2 t3.small (Jenkins) | ~$0.02/hr |
| EKS Cluster (control plane) | ~$0.10/hr |
| EC2 t3.small x2 (worker nodes) | ~$0.04/hr |
| ELB (LoadBalancer services) | ~$0.025/hr each |
| **Total** | **~$0.23/hr (~$5.50/day)** |

> 💡 **Tip:** Tear down the EKS cluster when not in use. You can recreate it in 15-20 minutes.

---

## Technologies Used

| Technology | Version | Purpose |
|-----------|---------|---------|
| Jenkins | Latest LTS | CI/CD pipeline orchestration |
| Docker | Latest | Container build and runtime |
| Amazon EKS | v1.34 | Managed Kubernetes |
| Prometheus | Latest (via Helm) | Metrics and alerting |
| Grafana | Latest (via Helm) | Dashboards and visualization |
| Maven | Latest | Java build and test |
| kubectl | Latest | Kubernetes CLI |
| eksctl | Latest | EKS cluster management |
| Helm | v3 | Kubernetes package manager |
| AWS CLI | v2 | AWS service management |

---

## Author

**Ibrahim** — DevOps Engineer

---

## License

This project is open source and available for learning purposes. Feel free to clone, modify, and use it in your own projects.