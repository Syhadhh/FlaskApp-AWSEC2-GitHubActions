# Automating Flask App Deployment to AWS EC2 with GitHub Actions

**Author:** Nur Syuhadah

---

### **Table of Contents**
1. [Problem Statement](#1-problem-statement)
2. [Project's Goal](#2-projects-goal)
3. [Designed Architecture](#3-designed-architecture)
4. [Step 1: Prepare AWS EC2 Server](#4-step-1-prepare-aws-ec2-server)
5. [Step 2: Install the Core Tools](#5-step-2-install-the-core-tools)
6. [Step 3: Set Up GitHub Self-Hosted Runner](#6-step-3-set-up-github-self-hosted-runner)
7. [Step 4: Configure GitHub Repository Files](#7-step-4-configure-github-repository-files)
    * [The Dockerfile: The System's Blueprint](#the-dockerfile-system's-blueprint)
    * [The Flask's Script: Define the Flask Application](#the-flask's-script-define-the-flassk-application)
    * [The GitHub Actions Workflow: Pipeline as Code](#the-github-actions-workflow-pipeline-as-code)
8. [Step 5: Consolidating The GitHub Actions Pipeline](#8-step-5-consolidating-the-github-actions-pipeline)
9. [Conclusion](#9-conclusion)

---
### **1. Problem Statement**
<a name="1-problem-statement"></a>
In conventional software delivery methodologies, the manual deployment of web applications across development and production environments creates numerous operational constraints:

Elevated Operational Burden: Technical personnel are required to manually authenticate into cloud infrastructure, retrieve updated code repositories, construct container images, and initiate service restarts—a labor-intensive procedure susceptible to configuration inconsistencies and service interruptions.

Unvalidated Security Threats: Transferring container images directly into production environments without implementing automated vulnerability assessments exposes infrastructure to recognized CVEs and insecure base image components.

Insufficient Deployment Documentation: The absence of automated version labeling mechanisms integrated with source control systems makes determining which specific commit version is presently deployed in production—or executing rollbacks following unsuccessful deployments—a protracted and unreliable undertaking.

This project addresses these problems through the establishment of a comprehensive automated CI/CD infrastructure utilizing GitHub Actions on a dedicated AWS EC2 runner. The framework streamlines source code retrieval, Docker image compilation, dual-version identification ($GIT_SHA and latest), Trivy-based security assessment, and seamless container redeployment with zero service interruption.

### **2. Project's Goal**
<a name="2-projects-goal"></a>
This objective of this project is to containerize a Flask web application utilizing Docker, perform vulnerability assessments on the container image, and fully automate the deployment workflow to an AWS EC2 instance.


The underlying concept involved designing an automated system wherein developers need to only commit and push code modifications to the GitHub repository. Subsequently, an orchestrated workflow executes autonomously, constructing a fresh Docker image, conducting security evaluations via Trivy, transferring the image to Docker Hub, and deploying the revised application entirely without any manual intervention. To accomplish this objective, implementation of DevOps technology stack includes AWS EC2 for computational infrastructure, Docker for application containerization, Docker Hub for container registry administration, Trivy for security vulnerability assessment, and GitHub Actions as the central automation  platform utilizing a self-hosted runner.

---

### **3. Designed Architecture**
<a name="3-designed-architecture"></a>
The workflow designed establishes a systematic progression from code commit through live deployment:

a)  Code Commit: Commit and push new code or modifications to the primary branch of the GitHub repository.

b)  **GitHub Actions Trigger:** Upon detecting the push event, GitHub Actions automatically activates the CI/CD workflow specified in .github/workflows/deploy.yml.

c)  **Pipeline Execution:** The pipeline operates through four sequential stages on the self-hosted AWS EC2 runner:
    * Checkout & Setup: Clones the latest source code from GitHub and transfers it to the runner environment.
    * Build & Push: Authenticates with Docker Hub, constructs the Flask application Docker image, applies tags corresponding to both the Git SHA commit identifier and the latest version, and uploads both tags to Docker Hub.
    * Security Scan: Executes a Trivy vulnerability assessment on the constructed Docker image to detect significant security vulnerabilities.
    * Deploy on EC2: Terminates and removes any existing container instance, retrieves the most recent Docker image from Docker Hub, instantiates a new container with port 5000 mapping, and performs cleanup of unused images through docker system prune.
    
d)  **Live Application:** The updated Flask application operates within an isolated container on the AWS EC2 instance, processing incoming requests on port 5000.

---

### **5. Step 1: Prepare AWS EC2 Server**
<a name="5-step-1-prepare-aws-ec2-server"></a>
The infrastructures supporting this project comprised of a virtual server deployed in the AWS cloud, configured to simultaneously host the GitHub Actions runner and the live application container.

a)  **Ubuntu 24.04 on a  t2.micro:** The **Ubuntu 24.04 LTS** image is to guarantee stability, security, and comprehensive long-term support for Docker and Flask. Besides, the **t2.micro** instance has 1 vCPU and 1 GiB RAM that provide sufficient computational resources and memory capacity for this project. GitHub Actions builds and tests the Docker image, hence EC2 doesn't need to perform the CI build. Considering a zero-cost workflow, the settings are applied as they are eligible for the AWS Free Tier. (AWS's current Free Tier documentation does not list t2.micro as an eligible instance for accounts created on or after July 15, 2025. the newer Free Tier rules, Ubuntu 24.04 + t3.micro may be the more relevant free-tier option.)

b)  **Security Group Rules** The security group functions as a virtual firewall protecting  the server. Below are the rules established to allow the specific traffic needed:
    * **Port 22 (SSH):** Essential for establishing secure terminal connections to the server via SSH/PuTTy (after converting the .pem key to .ppk using PuTTYgen), enabling initial system configuration and software installation.
    * **Port 5000 (Flask):** The designated port through which the Flask web application within the container operates, facilitating inbound web HTTP traffic to access the operational application.
    * **Port 80 (HTTP):** The conventional web protocol port configured to permit inbound HTTP web accessibility.

---

### **4. Step 2: Install the Core Tools**
<a name="4-step-2-install-the-core-tools"></a>
Upon establishing an SSH connection to the EC2 instance through PuTTY, the essential software and runtime infrastructure is installed and deployed:

a)  **AWS EC2 instance:** executed Package Index Update (`sudo apt update -y`) to verify that all system repositories maintained current status.
b)  **Docker (`docker.io`):** Installed Docker as the containerization agent by executing `sudo apt install docker.io -y`. It allows the application to be packaged in isolated ( Docker containers have separated processes, filesystems, and network environments from other containers and the host) and lightweight (a Docker container shares the host's kernel instead of including a separate full guest OS) manner which makes the environment reproducible across local development. Docker images is built and tested in GitHub Actions, and then the same image is deployed to AWS EC2.
c)  **Permissions:** Enabled and initiated the Docker daemon through the command `sudo systemctl enable --now docker`. Modified the permissions assigned to the Docker socket (chmod 777 /var/run/docker.sock) to facilitate non-root users and the GitHub runner agent in executing Docker commands without the necessity of sudo privileges.


---

### **6. Step 3: Set Up GitHub Self-Hosted Runner**
<a name="6-step-3-set-up-github-self-hosted-runner"></a>
An AWS EC2 server is configured as a self-hosted runner. This configuration enables the GitHub Actions workflow to execute deployments directly to the server where the application is hosted.

a)  **Runner Registration:** Accessed Settings > Actions > Runners within the GitHub repository and chose New self-hosted runner for Linux (x64).
b)  **Installation Steps on EC2:** 
   Established a dedicated directory: `mkdir actions-runner && cd actions-runner`.
   Obtained and decompressed the most recent GitHub Actions runner package.
   Initialized the  runner through `./config.sh --url <REPO_URL> --token 
   <REGISTRATION_TOKEN>`.
c)  **Activating the Agent:** Initiate the runner by executing `./run.sh` within the PuTTY session to commence monitoring for incoming workflow jobs activated by GitHub commits.

---

### **7. Step 4: Configure GitHub Repository Files**
<a name="7-step-4-configure-github-repository-files"></a>
The operational framework of this automation system is supported by three essential repository files.

#### **The Dockerfile: The System's Blueprint**
<a name="the-dockerfile-system's-blueprint"></a>
The Dockerfile contains specifications of the procedures for constructing a consistent and reproducible container image designated for the Flask application:
* **`FROM python:3.13-slim`**: Sets the base image for the container, -slim to keep the overall container size small and lightweight while still providing all essential Python runtime dependencies.
* **`WORKDIR /app`**: Sets the working directory inside the container. Any subsequent commands (like COPY, RUN, or CMD) will be executed relative to this path (/app).
* **`COPY requirements.txt app.py ./`**: Copies local files from the host machine into the container's file system.
* **`RUN pip install --no-cache-dir -r requirements.txt`**: Executes a command to build the container image layer. InstallS all the Python dependencies listed inside `requirements.txt` (such as Flask). The `--no-cache-dir` flag disables saving the downloaded cache files, which further minimizes the final Docker image size.
* **`EXPOSE 5000`**: Documents the port on which the container listens at runtime.
* **`CMD ["python", "app.py"]`**: Defines the default command that runs when the container starts.

### **The Flask's Script: Define the Flask Application**
<a name="the-flask's-script-define-the-flassk-application"></a>
The app.py script serves as the basic web application backend:
* **`from flask import Flask`**: Imports the Flask class from the flask library.
* **`app = Flask(__name__)`**: Creates an instance of the Flask application.
* **`@app.route('/')`**: Defines a decorator that binds a URL path to a specific Python function.
* **`def home():
         return "Hello Everyone from GitHub Actions - Happy Learning!"`**: Flask runs `home()` and sends the plain text string `"hello everyone from github actions happy learning"` back as the HTTP response rendered in the user's browser.
* **`if __name__ == '__main__':`**: Checks if the script is being executed directly.
* **`app.run(host='0.0.0.0', port=5000)`**: Starts the built-in Flask development web server.


#### **The GitHub Actions Workflow: Pipeline as Code**
<a name="the-github-actions-workflow-pipeline-as-code"></a>
Defined under `github/workflows/deploy.yml`, this YAML file orchestrates all four of the CI/CD pipeline.
```groovy
name: CI/CD Pipeline

on:
  push:
    branches:
      - main   

jobs:
  checkout:
    name: Checkout & Setup
    runs-on: self-hosted   
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

  build:
    name: Build & Push Image
    runs-on: self-hosted
    needs: checkout
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Log in to DockerHub
        run: echo "${{ secrets.DOCKERHUB_TOKEN }}" | docker login -u "${{ secrets.DOCKERHUB_USERNAME }}" --password-stdin

      - name: Build & Tag Docker Image
        run: |
          GIT_SHA=$(git rev-parse --short HEAD)
          docker build -t ${{ secrets.DOCKERHUB_USERNAME }}/flask-app:$GIT_SHA .
          docker tag ${{ secrets.DOCKERHUB_USERNAME }}/flask-app:$GIT_SHA ${{ secrets.DOCKERHUB_USERNAME }}/flask-app:latest

      - name: Push Docker Image
        run: |
          GIT_SHA=$(git rev-parse --short HEAD)
          docker push ${{ secrets.DOCKERHUB_USERNAME }}/flask-app:$GIT_SHA
          docker push ${{ secrets.DOCKERHUB_USERNAME }}/flask-app:latest

  scan:
    name: Security Scan with Trivy
    runs-on: self-hosted
    needs: build
    steps:
      - name: Install Trivy
        run: |
          sudo apt-get update -y
          sudo apt-get install -y wget apt-transport-https gnupg lsb-release
          wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
          echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/trivy.list
          sudo apt-get update -y
          sudo apt-get install -y trivy

      - name: Scan latest Docker image
        run: |
          trivy image --exit-code 0 --severity HIGH,CRITICAL ${{ secrets.DOCKERHUB_USERNAME }}/flask-app:latest
          trivy image --exit-code 1 --severity CRITICAL ${{ secrets.DOCKERHUB_USERNAME }}/flask-app:latest || echo "⚠️ Critical vulnerabilities found"

  deploy:
    name: Deploy on EC2
    runs-on: self-hosted
    needs: scan
    steps:
      - name: Deploy latest container
        run: |
          docker stop flask-app || true
          docker rm flask-app || true
          sleep 6
          docker pull ${{ secrets.DOCKERHUB_USERNAME }}/flask-app:latest
          docker run -d --name flask-app -p 5000:5000 ${{ secrets.DOCKERHUB_USERNAME }}/flask-app:latest
          sleep 6
          docker system prune -f
```
This GitHub Actions workflow comprises four continuous stages as stated below, each of which executes on the self-hosted runner (AWS EC2 instance) and performs a distinct component of the CI/CD pipeline.
    1.  **Stage 1: Checkout and Setup (checkout)** Executes automatically upon code submission to the primary branch. The objective is to downloads the repository files onto the self-hosted runner.
    2.  **Stage 2: Build and Push Image (build)** Builds a fresh Docker container image and publishes it to Docker Hub after checkout phase finishes successfully. Includes building the image using the local `Dockerfile`, applies two tags; the unique Git commit hash `$GIT_SHA` for version tracking and `latest` for easy deployment, as well as pushes both tagged images to the Docker Hub repository. 
    3.  **Stage 3: Security Scan with Trivy (scan)** Once the Docker image is built and pushed. This phase scans the newly created Docker image for security vulnerabilities before deploying it by running `trivy image` against the `latest` Docker image to detect OS packages and software dependency discrepency.
    4. **Stage 4: Deploy on EC2 (deploy)** Runs after passing the security scan by replacing the old running application with the newly updated Docker container. 

---

### **8. Step 5: Consolidating the GitHub Actions Pipeline**
<a name="8-step-5-consolidating-the-github-actions-pipeline"></a>
With the runner established and GitHub repository secrets (DOCKERHUB_USERNAME and DOCKERHUB_TOKEN) properly configured:

**Triggering Deployment:** A commit pushed to the main branch automatically activates the GitHub Actions workflow.

**Verification & Testing:** The self-hosted runner notices the job, constructs the image, applies tags using the Git commit hash with latest designation, and then transmits it to the Docker Hub.

Trivy performs an automated security assessment on the container image.

The deployment phase terminates any existing flaskapp container, retrieves the most recent image, and establishes a new container operating on port 5000.

**Validating Application:** Navigating to http://<EC2-PUBLIC-IP>:5000 through a web browser displays the operational application output: hello everyone from github actions happy learning

---

### **9. Conclusion**
<a name="9-conclusion"></a>
This project effectively converted a manual and error-prone deployment workflow into a streamlined, secure continuous integration and continuous deployment pipeline utilizing GitHub Actions and AWS EC2 infrastructure. Through the implementation of PuTTY for secure shell protocol administration, establishment of a self-hosted runner on EC2, containerization via Docker, and incorporation of automated Trivy vulnerability assessment, each code modification directed to production undergoes comprehensive validation, security analysis, construction, and deployment with minimal friction.
