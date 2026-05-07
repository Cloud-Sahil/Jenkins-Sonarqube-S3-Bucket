# Jenkins + SonarQube S3 Bucket 

---

## 1. Create EC2 Instances
- **Instance 1:** Jenkins (instance type : c7i-flex.large or more) 
- **Instance 2:** SonarQube  (instance type : c7i-flex.large or more)

---

## 2. Setup Jenkins EC2 Server

---
### Go To Root 
```sh
sudo -i
```

### Update Instance
```sh
apt update
```
### Install Java

### Install Maven
```bash
apt install maven -y
```
### Install Jenkins

Follow official Jenkins documentation
👉 [Jenkins Installation](https://www.jenkins.io/doc/book/installing/linux/)

### Install AWS Cli
```sh
snap install aws-cli --classic
```
### Jenkins 
```sh
su - jenkins
```
### Aws Configure

## 3. Setup SonarQube EC2 Server

### Go To Root 
```sh
sudo -i
```

### Update Instance
```sh
apt update
```
### Install Docker
```bash
apt install docker.io -y
```
### Run SonarQube
```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```
```sh
docker ps
```
## 4. Create S3 Bucket
- Name: `easycrud-artifact-bucket`
  
## 5. Access Jenkins & SonarQube
- Jenkins: `http://<jenkins-public-ip>:8080`
- SonarQube: `http://<sonarqube-public-ip>:9000`
    - Login SonarQube with:
        - Username: `admin`
        - Password: `admin`
        - Change the default password after first login.
          
## 6. Configure SonarQube

1. Create Webhook
    - Go to: **Administration** → **Configuration** → **Webhooks** → **Create**
    - Name: `Sonar-webhook`
    - URL: `http://<jenkins-public-ip>:8080/sonarqube-webhook/`
2. Create a Project
    - Go to **Projects** → **Create Project** → **Local Project*
    - Project display Name: `studentapp`
    - Project key: `studentapp`
    - Main branch name: `main` then click **Next**
    - Select **Use global settings**
    - Generate **token** → **Copy** & save it
    - Select **Maven** as build tool → Copy the given command

## 7. Configure Jenkins
### 1. Install Plugin
  - Dashboard → Manage Jenkins → Plugins → Available Plugins
  - Install: **SonarQube Scanner for Jenkins**
### 2. Add Credentials
  - Dashboard → Manage Jenkins → Credentials → Global credentials (unrestricted)
  - Add new credential:
      - Kind: `Secret Text`
      - Secret: `<SonarQube Token>`
      - ID: `sonar-token`
### 3. Configure SonarQube Server
  - Dashboard → Manage Jenkins → System
  - Find SonarQube Server section
  - Enable environment variable
  - Add new SonarQube:
      - Name: `Sonar-env`
      - Server URL: `http://<sonarqube-public-ip>:9000`
      - Authentication Token: `sonar-token`
  - Save changes
**Note (optional)**: After configuring, restart the Jenkins server to ensure it operates smoothly. (http://<jenkins-public-ip>:8080/restart)

## 8. Create Jenkins Pipeline
1. Go to Dashboard → New Item → Pipeline
2. Paste the following pipeline code:
```sh
pipeline {
    agent any

    stages {
        stage('clone repository') {
            steps {
                sh 'rm -rf EasyCRUD'
                git branch: 'main', url: 'https://github.com/Rohit-1920/EasyCRUD.git'
            }
        }

        stage('build') {
            steps {
                sh '''
                cd backend
                mvn clean package -DskipTests
                '''
            }
        }

        stage('sonar analysis') {
            steps {
                withSonarQubeEnv(credentialsId: 'sonar-token', installationName: 'Sonar-env') {
                    sh '''
                    cd backend
                    mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                      -Dsonar.projectKey=studentapp \
                      -Dsonar.projectName=studentapp
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('deploy') {
            steps {
                sh 'aws s3 cp backend/target/student-registration-backend-0.0.1-SNAPSHOT.jar s3://easycrud-artifact-bucket/easycrud.jar'
            }
        }
    }
}

```

## 9. Run the Pipeline
- Build the job in Jenkins.
- Check results in SonarQube Dashboard.
