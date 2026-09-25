# Automating Flask App Deployment to AWS EC2 with GitHub Actions

**Author:** Nur Syuhadah

---

### **Table of Contents**
1. [Project's Goal](#1-projects-goal)
2. [Designed Architecture](#2-designed-architecture)
3. [Step 1: Prepare AWS EC2 Server](#3-step-1-prepare-aws-ec2-server)
4. [Step 2: Install the Core Tools](#4-step-2-install-the-core-tools)
5. [Step 3: Set Up GitHub Self-Hosted Runner](#5-step-3-set-up-github-self-hosted-runner)
6. [Step 4: Configure GitHub Repository Files](#6-step-4-configure-github-repository-files)
    * [The Dockerfile: The System's Blueprint](#the-dockerfile-system's-blueprint)
    * [The Flask's Script: Define the Flask Application](#the-flask's-script-define-the-flassk-application)
    * [The GitHub Actions Workflow: Pipeline as Code](#the-github-actions-workflow-pipeline-as-code)
7. [Step 5: Bringing It All Together with a Jenkins Pipeline](#7-step-5-bringing-it-all-together-with-a-jenkins-pipeline)
8. [Final Thoughts and Conclusion](#8-final-thoughts-and-conclusion)

---

### **1. Project's Goal**
<a name="1-projects-goal"></a>
This objective of this project is to containerize a Flask web application utilizing Docker, perform vulnerability assessments on the container image, and fully automate the deployment workflow to an AWS EC2 instance.


The underlying concept involved designing an automated system wherein developers need to only commit and push code modifications to the GitHub repository. Subsequently, an orchestrated workflow executes autonomously, constructing a fresh Docker image, conducting security evaluations via Trivy, transferring the image to Docker Hub, and deploying the revised application entirely without any manual intervention. To accomplish this objective, implementation of DevOps technology stack includes AWS EC2 for computational infrastructure, Docker for application containerization, Docker Hub for container registry administration, Trivy for security vulnerability assessment, and GitHub Actions as the central automation  platform utilizing a self-hosted runner.

---

### **2. Designed Architecture**
<a name="2-designed-architecture"></a>
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

### **3. Step 1: Prepare AWS EC2 Server**
<a name="3-step-1-prepare-aws-ec2-server"></a>
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

### **5. Step 3: Set Up GitHub Self-Hosted Runner**
<a name="5-step-3-set-up-github-self-hosted-runner"></a>
An AWS EC2 server is configured as a self-hosted runner. This configuration enables the GitHub Actions workflow to execute deployments directly to the server where the application is hosted.

a)  **Runner Registration:** Accessed Settings > Actions > Runners within the GitHub repository and chose New self-hosted runner for Linux (x64).
b)  **Installation Steps on EC2:** 
   Established a dedicated directory: `mkdir actions-runner && cd actions-runner`.
   Obtained and decompressed the most recent GitHub Actions runner package.
   Initialized the  runner through `./config.sh --url <REPO_URL> --token 
   <REGISTRATION_TOKEN>`.
c)  **Activating the Agent:** Initiate the runner by executing `./run.sh` within the PuTTY session to commence monitoring for incoming workflow jobs activated by GitHub commits.

---

### **6. Step 4: Configure GitHub Repository Files**
<a name="6-step-4-configure-github-repository-files"></a>
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
* **`stages`**: I broke my pipeline into three logical stages:
    1.  **Clone Code:** Jenkins uses its Git plugin to clone my repository.
    2.  **Build Docker Image:** This step isn't strictly necessary since `docker compose` can also build, but I included it as an explicit step to make the process clearer.
    3.  **Deploy:** This is the magic step. `docker compose down || true` stops and removes any old running containers (the `|| true` prevents the pipeline from failing if there are no containers to stop). Then, `docker compose up -d --build` starts the application in the background, rebuilding the `flask` image to include the new code changes.

---

### **7. Step 5: Bringing It All Together with a Jenkins Pipeline**
<a name="7-step-5-bringing-it-all-together-with-a-jenkins-pipeline"></a>
With all the pieces in place, the final step was to create the pipeline job in Jenkins.
1.  I created a new "Pipeline" job in the Jenkins dashboard.
2.  Instead of writing the script in the text box, I configured it to pull the **"Pipeline script from SCM"**.
3.  I pointed it to my GitHub repository and told it the script file was named `Jenkinsfile`.

I then clicked **"Build Now"** to run the pipeline for the first time. I watched the logs in the "Console Output" as Jenkins cloned my code, built the image, and deployed the containers. After it finished, I was able to access my live application at `http://<my-ec2-ip>:5000`.

---

### **8. Final Thoughts and Conclusion**
<a name="8-final-thoughts-and-conclusion"></a>
This project was a fantastic journey through the core components of a modern DevOps workflow. I successfully built a fully automated CI/CD pipeline where a simple `git push` results in a live deployment. By containerizing the application with Docker, I've made it portable and consistent. By automating the process with Jenkins, I've made it fast, reliable, and repeatable. Any future changes to my application will now be deployed seamlessly, showcasing the true power of CI/CD.
