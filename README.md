# Student-ERP-CI-CD-with-Jenkins-Docker

# Student ERP — CI/CD with Jenkins & Docker

A CI/CD pipeline setup for deploying a **Student ERP** application (Spring Boot backend + frontend) on a single EC2 instance, using **Jenkins** for build/deploy automation, **Docker** for containerization, and **AWS RDS (MariaDB)** as the database.

---

## Project Overview

This project automates the build and deployment of a Student ERP system:

- **Backend**: Spring Boot application, connects to a MariaDB database hosted on AWS RDS.
- **Frontend**: Web app that connects to the backend via the EC2 instance's public IP.
- **CI/CD**: Jenkins builds Docker images for both frontend and backend, pushes them to Docker Hub, and deploys the containers on the EC2 instance.
- **Database**: AWS RDS (MariaDB) stores student records (`students` table — name, email, course, class, percentage, branch, mobile number).

**Port layout on the EC2 instance:**

| Service          | Port |
|------------------|------|
| Backend (app)     | 8080 |
| Jenkins           | 8081 (moved from default 8080 to avoid conflict) |

---

## Prerequisites

- An AWS account with permissions to launch EC2 and RDS instances.
- An EC2 instance: **c7i-flex.large**, Ubuntu-based AMI.
- Security group rules allowing:
  - `22` (SSH)
  - `8080` (backend app)
  - `8081` (Jenkins UI, after port change)
  - `3306` (MySQL/MariaDB, from EC2 → RDS)
- A Docker Hub account (for pushing built images).
- A GitHub repository containing the backend and frontend source code.
- Basic familiarity with Linux, Docker, and Jenkins pipelines.

---

## 1. Launch EC2 Instance

Launch one EC2 instance:
- Type: `c7i-flex.large`
- OS: Ubuntu

---

## 2. Install Dependencies on the Server

```bash
apt update -y

# Docker
apt install docker.io -y

# Java 21 (required for Jenkins)
apt install openjdk-21-jdk -y

# Jenkins (add repo key + source per official docs, then install)
apt install jenkins -y
```

---

## 3. Configure Jenkins

### 3.1 Access Jenkins
Open Jenkins in the browser at `http://<jenkins-ip>:8080` and complete the initial login/setup.

### 3.2 Change Jenkins Port (8080 → 8081)

Since the backend app also uses port `8080`, move Jenkins to `8081`:

```bash
nano /lib/systemd/system/jenkins.service
```

Update the `--httpPort` value to `8081`, then reload and restart:

```bash
systemctl daemon-reload
systemctl restart jenkins
```

### 3.3 Give Jenkins Docker Access

```bash
sudo usermod -aG docker jenkins
```

### 3.4 Give Jenkins Sudo Permissions

```bash
sudo visudo
```

Add the following line at the end of the file:

```
jenkins ALL=(ALL) NOPASSWD: ALL
```

### 3.5 Restart Jenkins

```
http://<jenkins-ip>:8081/restart
```

> ⚠️ Passwordless sudo for the `jenkins` user is convenient for a lab/demo setup but is a security risk in production — scope this down (e.g. limit to specific commands) for real deployments.

---

## 4. Create the Database (AWS RDS — MariaDB)

1. In the AWS Console, create an RDS instance with the **MariaDB** engine.
2. Choose **Full configuration** (not Easy Create) for more control.
3. Select the **security group** you normally use for this project (make sure it allows inbound `3306` from your EC2 instance).
4. Create the DB instance and note the **endpoint**.

### 4.1 Install MySQL Client on the EC2 Instance

```bash
apt install mysql-client -y
```

### 4.2 Connect to the Database

```bash
mysql -h <DB-endpoint> -u <username> -p
```

Enter the password when prompted.

### 4.3 Create Database and Grant Privileges

```sql
CREATE DATABASE student_db;
GRANT ALL PRIVILEGES ON student_db.* TO 'username'@'localhost' IDENTIFIED BY 'your_password';
```

> Replace `username` and `your_password` with your actual DB credentials — never commit real credentials to source control.

### 4.4 Select the Database

```sql
USE student_db;
```

### 4.5 Create the `students` Table

```sql
CREATE TABLE `students` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) DEFAULT NULL,
  `email` varchar(255) DEFAULT NULL,
  `course` varchar(255) DEFAULT NULL,
  `student_class` varchar(255) DEFAULT NULL,
  `percentage` double DEFAULT NULL,
  `branch` varchar(255) DEFAULT NULL,
  `mobile_number` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=80 DEFAULT CHARSET=latin1 COLLATE=latin1_swedish_ci;
```

Exit the MySQL shell:

```sql
exit
```

The database is now ready.

---

## 5. Configure the Application

### 5.1 Backend — `application.properties`

In the GitHub repo, edit:

```
backend/src/main/resources/application.properties
```

Set the RDS endpoint and DB credentials so the backend can connect to the database (datasource URL, username, password).

### 5.2 Frontend — `.env`

In the GitHub repo, edit the frontend's `.env` file and set the **EC2 instance's public IP** so the frontend can reach the backend API.

> Commit these as placeholders/environment variables where possible, rather than hardcoding real secrets into the repo.

---

## 6. Clean Up Existing Docker Resources

Before deploying, remove any old/unwanted containers, volumes, and images:

```bash
docker kill $(docker ps -q) && docker rm -v $(docker ps -a -q) && docker rmi $(docker images -q)
```

---

## 7. Add Docker Hub Credentials to Jenkins

Go to **Manage Jenkins → Credentials → Add Credentials**

| Field | Value |
|-------|-------|
| Kind      | Username with password |
| ID        | `dockerhub-cred` |
| Username  | `<dockerhub-username>` |
| Password  | `<dockerhub-password>` |

Click **Create**.

---

## 8. Write the Jenkins Pipeline

Create a Jenkins pipeline job that:
1. Checks out the backend and frontend code from GitHub.
2. Builds Docker images for both.
3. Pushes images to Docker Hub using the `dockerhub-cred` credential.
4. Pulls and runs the containers on the EC2 instance.

Example pipeline:

pipeline {
    agent any

    stages {
        stage('Clone Repo') {
            steps {
                git url: "https://github.com/SahilBhakare02/EasyCRUD-Updated.git", branch: "main"
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh "docker build -t iamsahil21/easycrud2-jenkins:frontend ./frontend"
            }
        }

        stage('Build Backend Image') {
            steps {
                sh "docker build --no-cache -t iamsahil21/easycrud2-jenkins:backend ./backend"
            }
        }

        stage('Docker Hub Login') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
                    sh "echo $PASSWORD | docker login -u $USERNAME --password-stdin"
                }
            }
        }

        stage('Push Frontend Image to Docker Hub') {
            steps {
                sh "docker push iamsahil21/easycrud2-jenkins:frontend"
            }
        }

        stage('Push Backend Image to Docker Hub') {
            steps {
                sh "docker push iamsahil21/easycrud2-jenkins:backend"
            }
        }

        stage('Run Frontend Container') {
            steps {
                sh '''
                    docker rm -f easycrud1-frontend || true
                    docker run -d --name easycrud1-frontend -p 80:80 iamsahil21/easycrud2-jenkins:frontend
                '''
            }
        }

        stage('Run Backend Container') {
            steps {
                sh '''
                    docker rm -f easycrud1-backend || true
                    docker run -d --name easycrud1-backend -p 8080:8080 iamsahil21/easycrud2-jenkins:backend
                '''
            }
        }
    }
}
```> When referencing the `dockerhub-cred` credential, Jenkins automatically exposes `_USR` and `_PSW` suffixed variables (e.g. `DOCKERHUB_CRED_USR`, `DOCKERHUB_CRED_PSW`) — make sure these are used consistently and never printed to logs.

---

## Security Notes

- Never hardcode DB passwords, Docker Hub passwords, or IPs directly into files committed to GitHub — use Jenkins credentials and environment variables.
- Passwordless sudo for the `jenkins` user should be restricted or removed outside of a lab/demo environment.
- Restrict RDS and EC2 security groups to only the necessary ports/IPs.
- Rotate any credentials that may have been shared or exposed during setup/testing.

---

## Summary

| Component  | Access |
|------------|--------|
| Jenkins    | `http://<jenkins-ip>:8081` |
| Backend    | `http://<ec2-public-ip>:8080` |
| Frontend   | `http://<ec2-public-ip>` (or configured port) |
| Database   | AWS RDS MariaDB endpoint (private, accessed from EC2 only) |

With this pipeline, every push triggers Jenkins to build fresh Docker images for the frontend and backend, push them to Docker Hub, and redeploy the containers on the EC2 instance.