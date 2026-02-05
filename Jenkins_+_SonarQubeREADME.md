# Jenkins + SonarQube CI Pipeline for Python Flask (AWS EC2)

This project demonstrates a complete **CI pipeline** for a Python Flask application using:

- GitHub (Source Code)
- Jenkins (CI Automation)
- SonarQube (Static Code Analysis)
- SonarScanner CLI
- AWS EC2 (Single server hosting all services)

---

## Server Architecture

All services run on the SAME EC2 instance

    ec2-user → SSH login user
    root     → Administrator (installs tools)
    sonar    → Runs SonarQube service
    jenkins  → Runs CI pipeline jobs

Important:
Jenkins does NOT read ~/.bashrc or user PATH variables.
Tools must be configured inside Jenkins UI.

---

## Project Repository

Repository used inside pipeline:

    https://github.com/CloudTechDevOps/docker_python_flask-project

Project structure after adding configuration:

    app.py
    requirements.txt
    sonar-project.properties
    Jenkinsfile

---

## Install SonarScanner (System Wide)

Download and extract:

    unzip sonar-scanner-cli-4.8.0.2856-linux.zip

Move to global directory:

    sudo mv sonar-scanner-4.8.0.2856-linux /opt/sonar-scanner
    sudo chmod -R 755 /opt/sonar-scanner
    sudo chown -R root:root /opt/sonar-scanner

Verify Jenkins can execute scanner:

    sudo -u jenkins /opt/sonar-scanner/bin/sonar-scanner -v

Expected:

    SonarScanner 4.8.0.2856

---

## Configure SonarScanner in Jenkins

Navigate:

    Manage Jenkins → Global Tool Configuration → SonarQube Scanner

Add configuration:

    Name: sonar-scanner
    Install automatically: unchecked
    SONAR_RUNNER_HOME: /opt/sonar-scanner

---

## Configure SonarQube Server in Jenkins

Navigate:

    Manage Jenkins → Configure System → SonarQube servers

Add server:

    Name: sonar
    Server URL: http://PRIVATE-IP:9000
    Token: generated from SonarQube → My Account → Security → Generate Token

---

## SonarQube Project Configuration

Create file:

    sonar-project.properties

Content:

    sonar.projectKey=flask-app
    sonar.projectName=flask-app
    sonar.projectVersion=1.0
    sonar.sources=.
    sonar.sourceEncoding=UTF-8
    sonar.python.version=3
    sonar.exclusions=venv/**,__pycache__/**,tests/**

Push to GitHub:

    git add sonar-project.properties
    git commit -m "Add SonarQube configuration"
    git push

---

## Jenkins Pipeline (Jenkinsfile)

The pipeline uses the repository URL directly:

    https://github.com/CloudTechDevOps/docker_python_flask-project.git

Pipeline definition:

    pipeline {
        agent any

        environment {
            SONARQUBE = 'sonar'
        }

        stages {

            stage('Checkout Code') {
                steps {
                    git branch: 'main', url: 'https://github.com/CloudTechDevOps/docker_python_flask-project.git'
                }
            }

            stage('Install Dependencies') {
                steps {
                    sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install -r requirements.txt || true
                    '''
                }
            }

            stage('Run Tests') {
                steps {
                    sh '''
                    . venv/bin/activate
                    pip install pytest || true
                    pytest || true
                    '''
                }
            }

            stage('SonarQube Analysis') {
                steps {
                    script {
                        def scannerHome = tool 'sonar-scanner'
                        withSonarQubeEnv('sonar') {
                            sh "${scannerHome}/bin/sonar-scanner"
                        }
                    }
                }
            }
        }
    }

---

## Running the Pipeline

Open Jenkins:

    http://EC2-PUBLIC-IP:8080

Trigger:

    Pipeline → Build Now

---

## Expected Successful Output

    Project root configuration file: sonar-project.properties
    SonarScanner 4.8.0.2856
    Analyzing on SonarQube server
    ANALYSIS SUCCESSFUL
    Dashboard: http://PRIVATE-IP:9000/dashboard?id=flask-app

Open SonarQube:

    http://EC2-PUBLIC-IP:9000

Your project will appear with metrics:

    Bugs
    Vulnerabilities
    Code Smells
    Coverage
    Duplications

---

## Common Errors & Solutions

### sonar-scanner: command not found
Cause:
Scanner installed in user home instead of global path

Fix:

    Move to /opt/sonar-scanner
    Configure in Jenkins Global Tool Configuration

---

### Use a tool from predefined Tool Installation
Cause:
Name mismatch

Fix:

    Jenkins tool name MUST match:
    tool 'sonar-scanner'

---

### Missing sonar.projectKey
Cause:
sonar-project.properties not created

Fix:
Create config file in repo root

---

### Permission denied sudoers
Cause:
Installing tools as sonar user

Fix:
Install software as root
Run services as service users

---

### Jenkins ignores PATH or .bashrc
Cause:
Jenkins runs as daemon

Fix:
Never rely on environment variables
Always configure tools in Jenkins UI

---

## Final Workflow

    Developer pushes code → GitHub
    Jenkins pulls code from https://github.com/CloudTechDevOps/docker_python_flask-project.git
    Creates Python virtual environment
    Installs dependencies
    Runs tests
    Executes SonarScanner
    Sends report to SonarQube
    SonarQube dashboard displays code quality

You now have a fully working CI quality pipeline 🚀
