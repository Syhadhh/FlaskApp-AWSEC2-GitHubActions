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
    * [The Dockerfile: A Blueprint for My App](#the-dockerfile-a-blueprint-for-my-app)
    * [The `docker-compose.yml`: Orchestrating My Services](#the-docker-composeyml-orchestrating-my-services)
    * [The Jenkinsfile: My Pipeline as Code](#the-jenkinsfile-my-pipeline-as-code)
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

2.  **Security Group Rules** The security group functions as a virtual firewall protecting  the server. Below are the rules established to allow the specific traffic needed:
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
b)  **Installation Steps on EC2:** Established a dedicated directory: `mkdir actions-runner && cd actions-runner`. Obtained and decompressed the most recent GitHub Actions runner package. Initialized the runner through `./config.sh --url <REPO_URL> --token <REGISTRATION_TOKEN>`.
c)  **Activating the Agent:** Initiated the runner by executing ./run.sh within the PuTTY session to commence monitoring for incoming workflow jobs activated by GitHub commits.

---

### **6. Step 4: Configure GitHub Repository Files**
<a name="6-step-4-configure-github-repository-files"></a>
The "brains" of my automation pipeline are three text files that I placed in my GitHub repository.

#### **The Dockerfile: A Blueprint for My App**
<a name="the-dockerfile-a-blueprint-for-my-app"></a>
The `Dockerfile` is a set of instructions for building a Docker image of my Flask application. It's like a recipe that ensures my application environment is identical every single time it's built.
* **`FROM python:3.9-slim`**: I started with an official, lightweight Python image.
* **`WORKDIR /app`**: This sets the working directory inside the container.
* **`RUN apt-get update ...`**: I installed some system libraries needed by the Python MySQL client.
* **`COPY requirements.txt .` & `RUN pip install ...`**: I copied the Python requirements file first and installed the dependencies. This is a clever optimization that uses Docker's caching. If my app code changes but my requirements don't, Docker doesn't need to re-install all the packages, making my builds much faster.
* **`COPY . .`**: This copies the rest of my application code into the image.
* **`CMD ["python", "app.py"]`**: This is the command that runs when the container starts.

#### **The `docker-compose.yml`: Orchestrating My Services**
<a name="the-docker-composeyml-orchestrating-my-services"></a>
This file is where I defined my entire 2-tier application.
* **`services:`**: I defined two services: `mysql` and `flask`.
* **`mysql:`**: This service uses the official `mysql` image from Docker Hub. I set environment variables for the database name and password. The `volumes` section is crucial: `mysql-data:/var/lib/mysql` creates a persistent volume. This means that even if I stop and remove the MySQL container, my data will not be lost.
* **`flask:`**: This service doesn't pull an image; it **builds** one using the `Dockerfile` in the current directory (`build: .`). I passed the database credentials to it as environment variables.
* **`depends_on:`**: This tells Docker to start the `mysql` container before it starts the `flask` container, which is essential since my app needs the database to be ready before it can connect.
* **`networks:`**: I created a custom network named `two-tier`. This allows the Flask and MySQL containers to find each other easily by their service names (`mysql`) instead of having to figure out their internal IP addresses.
* **`healthcheck:`**: This is a vital feature for reliability. Docker will periodically run these commands to ensure the containers are not just running, but are actually healthy and responsive.

#### **The Jenkinsfile: My Pipeline as Code**
<a name="the-jenkinsfile-my-pipeline-as-code"></a>
This file defines my CI/CD pipeline using Jenkins' "pipeline-as-code" syntax. Keeping the pipeline definition in my source code repository means my automation logic is version-controlled, just like my application code.
```groovy
pipeline {
    agent any
    stages {
        stage('Clone Code') {
            steps {
                git branch: 'main', url: '[https://github.com/your-username/your-repo.git](https://github.com/your-username/your-repo.git)'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t flask-app:latest .'
            }
        }
        stage('Deploy with Docker Compose') {
            steps {
                sh 'docker compose down || true'
                sh 'docker compose up -d --build'
            }
        }
    }
}
```
* **`agent any`**: This tells Jenkins it can run this pipeline on any available agent (in my case, the Jenkins server itself).
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
