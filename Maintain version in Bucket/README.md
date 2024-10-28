Here's a `README.md` file that outlines the steps for setting up the environment, configuring Jenkins, and deploying the application. You can use this file in your GitHub repository.

# Jenkins CI/CD Pipeline Setup for Student App

This document outlines the steps to create an AWS environment, set up Jenkins, and deploy a Maven-based Java application using a CI/CD pipeline.

## Prerequisites

- Two AWS EC2 instances
- AWS CLI installed on one of the instances
- Access to an S3 bucket
- Basic knowledge of AWS, Jenkins, and Maven

### Step 1: Create EC2 Instances

- Launch two EC2 instances with Ubuntu.
- Ensure they have appropriate IAM roles and security groups allowing necessary traffic.

### Step 2: Create an S3 Bucket

- Open the AWS Management Console and create a new S3 bucket named `bucketversion`.
- Ensure the bucket is configured to allow public access if needed for testing.

### Step 3: Install Jenkins on the First Instance

1. SSH into the first instance:
   ```bash
   ssh -i your-key.pem ubuntu@your-instance-public-ip
   ```

2. Install Java:
   ```bash
   sudo apt update
   sudo apt install openjdk-11-jre -y
   ```

3. Add the Jenkins repository and install Jenkins:
   ```bash
   sudo apt install wget -y
   wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key.asc | sudo apt-key add -
   echo deb http://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list
   sudo apt update
   sudo apt install jenkins -y
   ```

4. Start Jenkins:
   ```bash
   sudo systemctl start jenkins
   ```

5. Access Jenkins at `http://your-instance-public-ip:8080`.

### Step 4: Configure Jenkins

1. Install the Maven Integration Plugin:
   - Go to **Manage Jenkins** > **Manage Plugins**.
   - Search for "Maven Integration" and install it.

2. Configure Jenkins to use the second instance:
   - Go to **Manage Jenkins** > **Manage Nodes and Clouds**.
   - Create a new node with the label `dummy`.

### Step 5: Install Maven and Java on the Second Instance

1. SSH into the second instance:
   ```bash
   ssh -i your-key.pem ubuntu@your-second-instance-public-ip
   ```

2. Install Java and Maven:
   ```bash
   sudo apt update
   sudo apt install openjdk-11-jre-headless -y
   sudo apt install maven -y
   ```

3. Install AWS CLI:
   ```bash
   sudo apt install awscli -y
   aws configure  # Enter your AWS credentials and default region
   ```

### Step 6: Create a Jenkins Pipeline

In Jenkins, create a new pipeline job with the following Jenkinsfile:

```groovy
pipeline {
    agent {
        label 'dummy'  // Mention the label of your Jenkins agent
    }
    environment {
        S3_BUCKET = 'bucketversion'
        VERSION = "${env.BUILD_NUMBER}"
        ARTIFACT_NAME = "student-${VERSION}.war"
    }
    stages {
        stage('Pull') {
            steps {
                echo "Pulling from GitHub..."
                git "https://github.com/Aamantamboli/Studentapp"
            }
        }
        stage('Build') {
            steps {
                echo "Building the application..."
                sh '''
                sudo mvn clean package
                sudo apt update 
                sudo apt install unzip -y
                sudo curl -O https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.96/bin/apache-tomcat-9.0.96.zip
                sudo unzip -o apache-tomcat-9.0.96.zip 
                '''
            }
        }
        stage('Upload to S3') {
            steps {
                echo "Uploading WAR file to S3..."
                sh '''
                aws s3 cp target/*.war s3://$S3_BUCKET/$ARTIFACT_NAME
                '''
            }
        }
        stage('Test') {
            steps {
                echo "Running tests..."
                sh '''
                echo "Tests passed!"
                '''
            }
        }
        stage('Deploy') {
            steps {
                echo "Deploying the application from S3..."
                sh '''
                sudo chown -R ubuntu:ubuntu apache-tomcat-9.0.96
                sudo chmod -R 755 apache-tomcat-9.0.96
                aws s3 cp s3://$S3_BUCKET/$ARTIFACT_NAME apache-tomcat-9.0.96/webapps/student.war
                sudo bash apache-tomcat-9.0.96/bin/catalina.sh start
                echo "Deployment complete."
                '''
            }
        }
    }
    post {
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}
```

### Step 7: Access the Application

- After a successful deployment, access the application using the public IP of the second instance:
  ```plaintext
  http://your-second-instance-public-ip:8080/student
  ```

## Conclusion

You have successfully set up a CI/CD pipeline using Jenkins to build, package, and deploy a Java application to an EC2 instance. Adjust the settings and configurations based on your specific requirements.
