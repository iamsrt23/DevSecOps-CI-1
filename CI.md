# CI

### Production Grade DevSecOps  CICD Pipeline

**DEV ——>  QA  ——>  Staging ——> Production**

Continuous Delivery is a Practice where code changes are automatically prepares for a release to production. 

**CI/CD:**

1. Build Pipeline
2. Deployment Pipeline

CI is a practice that helps to frequently integrate code changes into a central repository and then automate Builds.

**Stages:**

1. Build and Unit Test

→ Generate Artifacts (.jar ,.war)

→ Unit test

We are Going to take All code so it is called as integrated test

Tools: Maven

1. Code Coverage

→ How many lines of code you tested

→ Unused Coce

Tools: Jacoco

1. Static Analysis or Software Composition analysis

→ Identify Vulnerabilities introduced by open-source or 3rd Party libraries used in code

Tools: OWASP Dependency-check 

1. Static Application Security Testing

→ Identify Vulnerabilities in proprietary Code

→ Insecure Coding Practices

Tools: Sonarqube

1. Quality Gates

→ Check if application meets the Quality standards

Tools: Sonarqube Quality Profile

1. Build Docker Image

→ Generate Deployable Arifact

Tools: Dockerfile

1. Scan Docker Image

→ Identify Vulnerabilities in Image Layers

Tools: Trivy

1. Smoke Test

→ Verify if the Image is built properly

→ Determine if Image/Application is ready for testing

Tools: Docker Container

### Pre-Req:

 1. Create 2 Ec2 servers

→ JenkinsMaster - t2.medium → install Jenkins in it.

→ Build Server -15gb Storage -t2.medium

→ Sonarqube -15gb Storage  -t2.medium

**Step 1: Ensure all the necessary plugins are installed in Jenkins Master**

- Parameterized trigger plugin
- Gitlab plugin
- Docker Pipeline
- Pipeline: AWS steps
- SonarQube Scanner
- Quality Gates

101806452690

**Build Machine:**

clone the Repo 

Create a File **setup.sh**

We can do with Ansible Also

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

```bash
#!/bin/bash

#---------------------------------------------#
# Author: Raviteja
#
#---------------------------------------------#

# Ensure script is run with root privileges
if [ "$EUID" -ne 0 ]; then
  echo "Please run this script as root or sudo privileges "
  exit 1
fi

# Install Java 8, Java 11 & Docker
apt update
apt install -y openjdk-8-jdk openjdk-11-jdk docker.io maven
usermod -a -G docker ubuntu

# Install Trivy
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
apt update
apt install -y trivy

sleep 5; clear
echo "   =================================="
echo "** Your Build server is ready for use **"
echo "   =================================="

```

Then Run the Command → sudo ./setup.sh

which java

which trivy

which docker

java —version

To Check the Java8 version—> ls  /usr/lib/jvm/java-8-openjdk-amd-64/bin/java —version

**Install Sonarqube on the t2.medium server:**

Run these commands Using sonarqube Container

```
$ sudo apt update
$ sudo apt install -y docker.io
$ sudo usermod -a -G docker ubuntu
$ sudo docker run -d --name sonar -p 9000:9000 sonarqube:lts-community

```

### **Generate Sonarqube token of type "global analysis token" and add it as Jenkins credential of type "secret text”:**

Login to Sonarqube   publicdns:9000

default:

username: admin

password: admin

change the password if required

Login—> Sonarqube server

My account —> Security —> Generate Global Analysis Token —> Copy the token

jenkins Publicdns:8080

Go to Jenkins—> ManageJenkins—> Credentials—> Global —>Add Credentials—>

 kind : Secret text,

Scope: Global

secret: sonar-token

ID: mysonarqube

**create**

sonarqube is a webapplication so Go System if it is a tool go for tool

ManageJenkins—> System—> SonarQube servers—>

Add Sonarqube:

Name: mysonarqube,

server Url: publicdns:9000

server authentication token: sonar-token 

**Apply**

### **Add dockerhub credentials as username/password type:**

Jenkins Publicdns:8080

Go to Jenkins—> ManageJenkins—> Credentials—> Global —>Add Credentials—>

kind: username and password,

Scope: Global,

username: my dockerhub username,

password: my dockerhub password,(otherwise using secret text add docker hub password)

ID: dockerhub

Generate webhook & add the Jenkins URL as follows -

 [http://JenkinspublicDNS:8080/sonarqube-webhook/](http://url:8080/sonarqube-webhook/)

Goto:

SonarQubePublicDNS:9090

Administration —> configuration —> Webhooks —> Create

Name: demo

URL: [http://JenkinspublicDNS:8080/sonarqube-webhook/](http://url:8080/sonarqube-webhook/) 

**create**

for OWASP Dependency checks in sonarqube

Administration —> marketplace —> install of plugins (I understand the risk)

Plugins search—> dependency-check integration (install)

Restart Server

**Add Build server credentials for Jenkins master to connect:**

 Go to Jenkins—> ManageJenkins—> Credentials—> Global —>Add Credentials—>

kind: ssh username and privatekey,

Scope: Global,

username: ubuntu,

private Key: 

Enter Directly —> Add

copy the pem file content

(cat my.pem)

**ADD**

ManageJenkins —> Manage Nodes  —> New Node —>

Node name: Buildserver

Type: Permanent Agent

**create**

Number of Executors: 1

Remote Root Directory: /tmp/jenkins

Labels: build

Usage: Only build jobs with label expression matching this node

Launch method: Launch agents via ssh

Host: Build server privateip

Credentials: Add Ubuntu

Host Key Verification Stratagey: Manually Trusted Verification Strategy

**SAVE**

Node Will be Created with name Build Server Click on it Check the Logs (Master is Connected or not)

**Add Github credentials:**

jenkins Publicdns:8080

Go to Jenkins—> ManageJenkins—> Credentials—> Global —>Add Credentials—>

 kind : Secret text,

Scope: Global

secret: githubtoken

ID: githubcred

**create**

**Jenkinsfile:**

In sonarQube —> Quality Gate 

```groovy
pipeline{
    agent {label 'build'}
    environment {
        registry = "iamsrt23/democicd"
        registryCredential = "dockerhub"
    }
    stages{
        stage('Checkout'){
            steps{
            git branch: 'main', credentialsId: 'githubtoken', url: 'https://github.com/iamsrt23/DevSecOps-CICD-1.git'

            }
        }
        stage('Stage1: Build'){
            steps{
                echo "Building Jar Component ..."
                sh "export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64; mvn clean package"
                
            }
        }
        stage('Stage2: Code Coverage'){
            steps{
                echo "Running Code Coverage ...."
                sh  "export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64; mvn jacoco:report "
                
            }
        }
        stage('Stage3 : SCA'){
            steps{
                echo "Running Software Composition Analysis"
                sh  "export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64; mvn org.owasp:dependency-check-maven:check"
                
            }
        }
        stage('Stage4: SAST'){
            steps{
                echo "Running Static Applicatio security testing using SonarQube Scanner...."
                withSonarQubeEnv('mysonarqube'){
                    sh "mvn sonar:sonar -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml -Dsonar.dependencyCheck.jsonReportPath=target/dependency-check-report.json -Dsonar.depencencyCheck.htmlReportPath=target/dependency-check-report.html -Dsonar.projectName='cicd-demo'"
                }
                

            }
        }
        stage('Stage 5: QualityGates'){
            steps{
                echo "Running Quality Gates to verify the code quality"
                script{
                    timeout(time: 1, unit: 'MINUTES'){
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK'){

                            error 'Pipeline aborted due to quality gate failure: ${qg.status}'

                        }
                    }
                }
                
            }
        }
        stage('Stage6: BuildImage'){
            steps{
                echo "Build Docker Image"

                script{
                    docker.withRegistry('',registryCredential){
                        myImage = docker.build registry
                        myImage.push()
                    }
                }

                
            }
        }
        stage('Stage 7: Scan Image'){
            steps{
                echo "Scanning Image for Vulnerabilities"
                sh "trivy image --scanners vuln --offline-scan iamsrt23/democicd:latest > trivyresults.txt"

                
            }
        }
        stage('Stage 8: SmokeTest'){
            steps{
                echo "Smoke test the Image"
                sh "docker run -d --name smokerun -p 8000:8000 iamsrt23/democicd"
                sh "sleep 90; ./check.sh"
                sh "docker rm --force smokerun"
            }
        }
    }
}
```

Add GitHub Crednentials

Create a Webhook for project:

project —> Settings —> webhook —> http://<JENKINS_URL>/github-webhook/

- **Content Type**: Select application/json.

•	**Secret**: (Optional) Provide a secret token for secure communication. If used, configure this in Jenkins.

•	**Events**: Select the events you want the webhook to trigger on (e.g., **Push events**).

•	Click **Add Webhook**.

In Jenkins —> pipeline job —> •	Check **GitHub hook trigger for GITScm polling**.

**Up to Here We done CI pipeline**