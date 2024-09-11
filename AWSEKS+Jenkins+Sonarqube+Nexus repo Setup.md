# Jenkins CICD End to End Setup for a Java Based Application on AWS

## Requirements

### Step 1: Create Security Group
Create a Security Group for all 3 servers and open the following ports:

| Type            | Protocol | Port Range         | Source        |
|-----------------|----------|--------------------|---------------|
| Custom UDP      | UDP      | 88                 | 0.0.0.0/0     |
| SMTP            | TCP      | 25                 | 0.0.0.0/0     |
| DNS (UDP)       | UDP      | 53                 | 0.0.0.0/0     |
| Oracle-RDS      | TCP      | 1521               | 0.0.0.0/0     |
| Custom UDP      | UDP      | 389                | 0.0.0.0/0     |
| Custom TCP      | TCP      | 8080               | 0.0.0.0/0     |
| Custom TCP      | TCP      | 6443               | 0.0.0.0/0     |
| DNS (TCP)       | TCP      | 53                 | 0.0.0.0/0     |
| Custom TCP      | TCP      | 88                 | 0.0.0.0/0     |
| Custom TCP      | TCP      | 9000               | 0.0.0.0/0     |
| Custom TCP      | TCP      | 30000 - 32767      | 0.0.0.0/0     |
| HTTPS           | TCP      | 443                | 0.0.0.0/0     |
| SMTPS           | TCP      | 465                | 0.0.0.0/0     |
| Custom TCP      | TCP      | 30950              | 0.0.0.0/0     |
| Custom TCP      | TCP      | 3000 - 10000       | 0.0.0.0/0     |
| Custom TCP      | TCP      | 8025               | 0.0.0.0/0     |
| HTTP            | TCP      | 80                 | 0.0.0.0/0     |
| SSH             | TCP      | 22                 | 0.0.0.0/0     |
| Custom TCP      | TCP      | 8081               | 0.0.0.0/0     |
| LDAP            | TCP      | 389                | 0.0.0.0/0     |

### Step 2: Create 3 Ubuntu Machines
- Use `t2.medium` instances (2 CPUs, 4GiB RAM, 25GB Storage)
- Use the same keypair and security group for better accessibility

The 3 servers:
1. **Jenkins** - For CICD
2. **SonarQube** - For code quality checking
3. **Nexus Repo** - For storing build artifacts

### Step 3: Install Jenkins
**On Server 1 (Jenkins Server):**
```bash
# Update system and install OpenJDK
sudo apt-get update
sudo apt-get install openjdk-17-jre-headless -y

# Add Jenkins repository and install Jenkins
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
```

Install Docker on Jenkins Server using below commands
```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo chmod 666 /var/run/docker.sock
docker --version
```
#Change the Docker sock file permission in jenkins server so that jenkins would be able to run docker commands from anywhere
```bash
sudo chmod 666 /var/run/docker.sock
```
Jenkins Installation has been done ,now we need to configure the jenkins 

# Jenkins, SonarQube, and Nexus Setup Guide

## Step 1: Access Jenkins Server

1. Access Jenkins using the public IP of the Jenkins server: `<Jenkins-Server-IP>:8080`.
2. Jenkins will prompt for the initial admin password. Retrieve it from the Jenkins server using:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
   If access is denied, change the file permission. 
3. Copy the password and paste it into the Jenkins web UI. 
4. Set up Jenkins with a username, password, and email of your choice. 
5. During the setup, Jenkins will ask to install plugins. Choose the plugins based on your requirements and wait for the installation.

## Plugin Installation
  Go to Manage Jenkins → Manage Plugins.On the left side, click on Available Plugins.Install the following plugins:
- Eclipse Temurin Installer
- Maven (for Maven install - Config File Provider)
- Maven Integration Pipeline
- SonarQube Scanner
- Docker Credentials
- Credentials Manager
- Kubernetes
- Kubernetes CLI
- Kubernetes Client API
- Multibranch Pipeline
- Workspace Cleanup
- Docker Pipeline
- Stage View . Wait for all plugins to be installed and then return to the Jenkins Dashboard.

## Step 2: SonarQube Installation and Setup
Install Docker on SonarQube Server
```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo chmod 666 /var/run/docker.sock
docker --version
```
Pull and Run SonarQube Image
Pull the SonarQube image and run it in a Docker container:
```bash
docker run -d --name Sonar -p 9000:9000 sonarqube:lts-community
```

1. Access SonarQube using the public IP of the SonarQube server:
http://<SonarQube-Server-IP>:9000
2. The default credentials are:
- **Username:** admin
- **Password:** admin

## Generate SonarQube Token

1. Navigate to:
Administration → Security → Users

2. Select the admin user and generate a token for API access.

3. Copy the token and add it to Jenkins as a credential:
- Go to:
  ```
  Manage Jenkins → Credentials → Global → Add Credentials
  ```
- Choose **Secret Text** for the kind and paste the generated token.
- Give it an ID: `sonar-token`.

## SonarQube Webhook

1. Go to:
Administration → Configuration → Webhooks

2. Create a webhook with the following details:
- **Name:** CICD
- **URL:** `http://<Jenkins-Server-IP>:8080/sonarqube-webhook/`

## Step 3: Nexus Repository Installation and Setup

1. Install Docker on the Nexus Server by following the same Docker installation steps as for the SonarQube setup.

2. After installation, run the Nexus container.
```bash
docker run -d --name Nexus -p 8081:8081 sonatype/nexus3
```
Access Nexus using the public IP of the Nexus server: <Nexus-Server-IP>:8081.
- Retrieve the default admin password from the Nexus container:
```bash
docker exec -it <container-id> /bin/bash
```
- Reset the password and log in with the new credentials.

## Step 4: Configure Jenkins for SonarQube and Nexus

- Go to Manage Jenkins → System Configuration.
1. In SonarQube Servers: Name: sonar ,  URL: http://<SonarQube-Server-IP>:9000, Server Authentication Token: sonar-token
2. For Email Notification
- Add Gmail credentials in Jenkins(gmail as userid and passwd: app passwd) (create app password from Google account).
- kind : userid and password 
- user : usergmail 
- passwod : paste it
- id:mail-cred
- description:mail cred got to Systems- Extended email notifications:
- SMTP Server: smtp.gmail.com
- SMTP Port: 465
- Use SSL: Yes
```bash
now Section-Extended E-mail Notification
Extended E-mail Notification
SMTP server
smtp.gmail.com
SMTP Port
465
Advanced
Edited
Credentials

jeevanclk712000@gmail.com/****** (mail-cred)

checkbox sould be checked for Use SSL


E-mail Notification section 
SMTP server
smtp.gmail.com

Advanced
Edited

Use SMTP Authentication checkbox should be checked
User Name
jeevanclk712000@gmail.com
Password
•••••••••••••••••••

Use SSL :checkbox should be checked
```
3. For Docker:
- Add DockerHub credentials in Jenkins.

## Step 5: Configure Global Tools in Jenkins
- Go to Manage Jenkins → Global Tool Configuration.
Configure the following tools:
#
JDK:
Name: jdk17
Automatically install JDK from adoptium.net.
#
SonarQube Scanner:
Name: sonar-scanner
Automatically install SonarQube Scanner.
#
Maven:
Name: maven3
Automatically install Maven version 3.9.6.
#
## Step 6: Nexus Repository Configuration in Jenkins
Go to Manage Jenkins → Managed Files.
Add new configuration:
Global Maven settings.xml
ID: global-settings
Add the following content to connect to Nexus:
```bash
<servers>
  <server>
    <id>maven-releases</id>
    <username>admin</username>
    <password>your-nexus-password</password>
  </server>
  <server>
    <id>maven-snapshots</id>
    <username>admin</username>
    <password>your-nexus-password</password>
  </server>
</servers>
```
## Step 7: Update pom.xml for Nexus Repository
Add the following repository configuration to your pom.xml file:

```bash
<distributionManagement>
  <repository>
    <id>maven-releases</id>
    <url>http://<Nexus-Server-IP>:8081/repository/maven-releases/</url>
  </repository>
  <snapshotRepository>
    <id>maven-snapshots</id>
    <url>http://<Nexus-Server-IP>:8081/repository/maven-snapshots/</url>
  </snapshotRepository>
</distributionManagement>
```
## Step 8: Create Jenkins Pipeline Job
In Jenkins, create a new pipeline job:
Name: cicd.
Choose Pipeline project.
Configure the pipeline script based on your project and tools.
You are now ready to run the CI/CD pipeline with SonarQube and Nexus integration!

paste the below pipeline in the pipeline section
```bash
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Git Checkout') {
            steps {
                parallel(
                    ultimate: {
                        dir('ultimate') {
                            git changelog: false, poll: false, url: 'https://github.com/JeevanCLK/BoardGame.git'
                        }
                    }
                )
            }
        }
        stage('Code Compile') {
            steps {
                parallel(
                    ultimate: {
                        dir('ultimate') {
                            sh 'mvn compile'
                        }
                    }
                )
            }
        }
        stage('Run Test Cases') {
            steps {
                parallel(
                    ultimate: {
                        dir('ultimate') {
                            sh 'mvn test'
                        }
                    }
                )
            }
        }
        stage('File System Scan by Trivy') {
            steps {
                parallel(
                    ultimate: {
                        dir('ultimate') {
                            sh 'trivy fs --format table -o trivy-fs-report.html src'
                        }
                    }
                )
            }
        }
        stage('SonarQube Code Analysis') {
            steps {
                parallel(
                    ultimate: {
                        dir('ultimate') {
                            withSonarQubeEnv('sonar') {
                                sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Ultimate -Dsonar.projectKey=Ultimate -Dsonar.java.binaries=.'''
                            }
                        }
                    }
                )
            }
        }
        stage('Quality Gate by SonarQube') {
            steps {
                parallel(
                    ultimate: {
                        dir('ultimate') {
                            script {
                                waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                            }
                        }
                    }
                )
            }
        }
        stage('Build Application') {
            steps {
                parallel(
                    ultimate: {
                        dir('ultimate') {
                            sh 'mvn package'
                        }
                    }
                )
            }
        }
        stage('Publish Artifacts to Nexus Repo') {
            steps {
                parallel(
                    ultimate: {
                        dir('ultimate') {
                            withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                                sh 'mvn deploy'
                            }
                        }
                    }
                )
            }
        }
        stage('Docker Build and Tag Images') {
            steps {
                parallel(
                    ultimate: {
                        dir('ultimate') {
                            script {
                                withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                                    sh 'docker build -t DockerHubUserID/NameoftheImage:latest .'
                                }
                            }
                        }
                    }
                )
            }
        }
        stage('Image Scan by Trivy') {
            steps {
                parallel(
                    ultimate: {
                        dir('ultimate') {
                            sh 'trivy image --format table -o trivy-fs-report.html DockerHubUserID/NameoftheImage:latest'
                        }
                    }
                )
            }
        }
        stage('Push Image to DockerHub') {
            steps {
                parallel(
                    ultimate: {
                        dir('ultimate') {
                            script {
                                withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                                    sh 'docker push DockerHubUserID/NameoftheImage:latest'
                                }
                            }
                        }
                    }
                )
            }
        }
        stage('Deploy to AWS EKS') {
            steps {
                parallel(
                    ultimate: {
                        dir('ultimate') {
                                           withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'EKS-1', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', serverUrl: 'https://9F39F577334FF23706994135261985F2.gr7.ap-south-1.eks.amazonaws.com']]) {
                                          sh "kubectl apply -f deployment-service.yml"
                         }
                     }
                 )
             }
         }
        }
        stage('Verify the Deployment') {
            steps {
                parallel(
                    ultimate: {
                        dir('ultimate') {
                           withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'EKS-1', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', serverUrl: 'https://9F39F577334FF23706994135261985F2.gr7.ap-south-1.eks.amazonaws.com']]) {
                    sh "kubectl get svc -n webapps"
                    }
                        }
                    }
                )
            }
        }
    }
    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

               def body = """
                    <html>
                    <body>
                    <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                    <h2>${jobName} - Build ${buildNumber}</h2>
                    <div style="background-color: ${bannerColor}; padding: 10px;">
                    <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                    </div>
                    <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                    </div>
                    </body>
                    </html>
                """
                
                emailext(
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'YourGmainID@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'ultimate/trivy-fs-report.html'
                )
            }
        }
    }
}
```
- Build Job to execute pipeline
- The pipeline should run successfully until it pushes the Image to Docker , post that we need to Create AWS EKS cluster and need to create compute resources and roles and then we need add policies to the roles created.
- Connect to AWS EKS cluster and create Service accound, create role for the service account and then we need to bind the role to the service account .


## First Create a user in AWS IAM with any name
## Attach Policies to the newly created user
## below policies
AmazonEC2FullAccess

AmazonEKS_CNI_Policy

AmazonEKSClusterPolicy	

AmazonEKSWorkerNodePolicy

AWSCloudFormationFullAccess

IAMFullAccess

#### One more policy we need to create with content as below
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "eks:*",
            "Resource": "*"
        }
    ]
}
```
Attach this policy to your user as well

![Policies To Attach](https://github.com/jaiswaladi246/EKS-Complete/blob/main/Policies.png)

# AWSCLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
aws configure
```

## KUBECTL

```bash
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
```

## EKSCTL

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

## Create EKS CLUSTER

```bash
eksctl create cluster --name=my-eks22 \
                      --region=ap-south-1 \
                      --zones=ap-south-1a,ap-south-1b \
                      --version=1.30 \
                      --without-nodegroup

eksctl utils associate-iam-oidc-provider \
    --region ap-south-1 \
    --cluster my-eks22 \
    --approve

eksctl create nodegroup --cluster=my-eks22 \
                       --region=ap-south-1 \
                       --name=node2 \
                       --node-type=t3.medium \
                       --nodes=3 \
                       --nodes-min=2 \
                       --nodes-max=4 \
                       --node-volume-size=20 \
                       --ssh-access \
                       --ssh-public-key=Key \
                       --managed \
                       --asg-access \
                       --external-dns-access \
                       --full-ecr-access \
                       --appmesh-access \
                       --alb-ingress-access
```

* Open INBOUND TRAFFIC IN ADDITIONAL Security Group
* Create Servcie account/ROLE/BIND-ROLE/Token

## Create Service Account, Role & Assign that role, And create a secret for Service Account and geenrate a Token

### Creating Service Account


```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: webapps
```

### Create Role 


```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: webapps
rules:
  - apiGroups:
        - ""
        - apps
        - autoscaling
        - batch
        - extensions
        - policy
        - rbac.authorization.k8s.io
    resources:
      - pods
      - secrets
      - componentstatuses
      - configmaps
      - daemonsets
      - deployments
      - events
      - endpoints
      - horizontalpodautoscalers
      - ingress
      - jobs
      - limitranges
      - namespaces
      - nodes
      - pods
      - persistentvolumes
      - persistentvolumeclaims
      - resourcequotas
      - replicasets
      - replicationcontrollers
      - serviceaccounts
      - services
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

### Bind the role to service account


```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rolebinding
  namespace: webapps 
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-role 
subjects:
- namespace: webapps 
  kind: ServiceAccount
  name: jenkins 
```

### Generate token using service account in the namespace

[Create Token](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#:~:text=To%20create%20a%20non%2Dexpiring,with%20that%20generated%20token%20data.)

