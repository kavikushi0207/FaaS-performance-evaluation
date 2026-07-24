Resume/JD Matcher API: A FaaS Performance Evaluation

# 📌 Project Overview

This project implements a Serverless HTTP API designed to analyze resumes against job descriptions (JD). It utilizes Natural Language Processing (NLP) to return a match score and identifies "skill gaps" to provide career insights. The core objective of this study is to benchmark Function-as-a-Service (FaaS) behaviors, specifically focusing on cold vs. warm start latency, horizontal scaling, and resource management within a Kubernetes environment.

# 🏗 System Architecture

The system follows a containerized microservices pattern:


* Serverless API: Developed using Azure Functions (Python V2).


* NLP Engine: Built with Scikit-learn using TF-IDF Vectorization and Cosine Similarity.


* Containerization: Packaged via Docker (emulated for linux/amd64 compatibility).
* Cloud Deployment: Deploying the containerized image into Azure Container Registry



* Automated Orchestration: Managed by KEDA inside Azure Container Registry
* Testing with JMeter to evaluate the FaaS performance under variant criteria

<img width="1570" height="614" alt="sys-arch" src="https://github.com/user-attachments/assets/dc20a989-7a76-4d34-b8f4-2390c262d2c5" />


# 🚀 Getting Started

 ## Prerequisites
 
* Docker Desktop 
* Minikube & kubectl (for local testing)
* Azure Functions Core Tools
* Azure Cloud Services
  1. Resource Group
  2. Container Registry (and KEDA inside for auto-scaling)
  3. Container App
  4. Virtual Machine
* Jmeter

## Deployment Steps into Docker

1. Build the Image:

   ```
    docker build --platform linux/amd64 -t resume-matcher-v1 .
   ```
   
3. Initialize Cluster:
   ```
    minikube start --driver=docker
    minikube image load resume-matcher-v1
   ```
5. Deploy to Kubernetes:
   ```
    kubectl apply -f deployment.yaml
   ```
7. Access the API:

   ```
    minikube service resume-matcher-service --url
   ```
## Deployment to the Azure cloud

### 1. Azure Container Registry (ACR) Deployment

The first step is to move the locally built Docker image (`resume-matcher-v1`) into the Azure cloud. We tagged the local image for our Azure registry, authenticated, and pushed the image.

**Tag the local image for the Azure registry:**
```
docker tag resume-matcher-v1 polimiregistry.azurecr.io/resume-matcher:v1
```
**Authenticate with ACR:**
```
docker login polimiregistry.azurecr.io -u polimiregistry -p <ACR_PASSWORD>
```
**Push the image to the cloud:**
```
docker push polimiregistry.azurecr.io/resume-matcher:v1
```

**To obtain credentials of ACR:**
```
az acr credential show
```
### 2. Provisioning the Cloud Infrastructure

Instead of manually configuring a Kubernetes cluster, we utilized Azure Container Apps. First, we created the managed environment (the underlying network and cluster), and then deployed the Container App with built-in KEDA autoscaling rules (scaling from 1 to 10 replicas).

### Create the Azure Container Apps Environment

```

az containerapp env create \
  --name polimi-env \
  --resource-group faas_project \
  --location francecentral
```
### Create and deploy the Container App
```
az containerapp create \
  --name resumematcher-app \
  --resource-group faas_project \
  --environment polimi-env \
  --image polimiregistry.azurecr.io/resume-matcher:v1 \
  --registry-server polimiregistry.azurecr.io \
  --registry-username polimiregistry \
  --registry-password <ACR_PASSWORD> \
  --ingress external \
  --target-port 80 \
  --min-replicas 1 \
  --max-replicas 10
```
### 3. Deploying Code Updates (v2)
After enhancing the NLP model, we needed to deploy the new code without causing downtime. We achieved this by building a v2 image and issuing an update command to Azure, which safely rolled out the new containers.

#### Build and push the new image version
```
docker build --platform linux/amd64 -t polimiregistry.azurecr.io/resume-matcher:v2 .
docker push polimiregistry.azurecr.io/resume-matcher:v2
```
#### Update the running Container App with the new image
```
az containerapp update \
  --name resumematcher-app \
  --resource-group faas_project \
  --image polimiregistry.azurecr.io/resume-matcher:v2
  ```
#### Test run of docker image:
```
curl -X POST https://<FQDN>/api/match \
  -H "Content-Type: application/json" \
  -d '{"resume": "Software engineer with 5 years of experience in Python, Azure, and Docker.", "jd": "Looking for a backend developer skilled in Python and cloud infrastructure."}'
```
#### Our FQDN: 
```
resumematcher-app.redsmoke-cd88e19a.francecentral.azurecontainerapps.io
```
###  4. Performance Benchmarking & Evaluation
 
We utilized Apache JMeter to simulate concurrent users and evaluate the system's reliability and scalability. The evaluation focused on three primary test scenarios to analyze the behavior of the FaaS architecture.

#### Installing JMeter on Mac (for creating tests in GUI mode)
```
sudo install jmeter 
```
#### Open the software
```
jmeter
```
---
### 📊 JMeter Performance Testing Setup
## Performance Evaluation Strategy

Following the benchmarking principles from the computing infrastructure course (Polimi, Prof. Danilo Ardagna), our testing strategy moves beyond simple baseline testing. We evaluate the Azure serverless application using the following scenarios, visualizing all metrics through the JMeter HTML Dashboard:

* **Cold vs. Warm Starts:** Measuring the initial latency overhead when the Azure container scales from zero, compared to the execution time of an already provisioned instance.
* **Load & Stress Testing:** Incrementally increasing concurrent users to identify the system's saturation point, throughput limits, and degradation curve.
* **Spike Testing:** Simulating sudden, massive surges in traffic to evaluate how rapidly the FaaS autoscaler provisions new instances.
* **Payload & Mixed Workload Testing:** Varying the size of the JSON request body and combining different API request patterns to mimic real-world usage.
* **Pareto Analysis:** Using the generated dashboard data to identify the 20% of performance bottlenecks that are causing 80% of the latency or failure issues.

---

## Setting up JMeter on the Azure VM

To ensure accurate latency measurements and avoid local network jitter, a dedicated testing VM is deployed inside the Azure Resource Group. 

Once the standard Ubuntu Linux VM is running, SSH into it and execute the following commands to prepare the environment:

```bash
# 1. Update packages and install Java (Required for JMeter)
sudo apt-get update
sudo apt-get install -y default-jre

# 2. Download and extract the latest JMeter binary
wget [https://dlcdn.apache.org//jmeter/binaries/apache-jmeter-5.6.3.tgz](https://dlcdn.apache.org//jmeter/binaries/apache-jmeter-5.6.3.tgz)
tar -xf apache-jmeter-5.6.3.tgz

# 3. Add JMeter to the system path for easier execution
export PATH=$PATH:~/apache-jmeter-5.6.3/bin
```

# 🛠 Troubleshooting & Insights

* Auth Level: Set to ANONYMOUS for localized benchmarking to eliminate security handshake overhead.

* Resource Constraints: Identified that emulated x86_64 environments on ARM64 hosts require higher CPU/RAM quotas to prevent SIGABRT errors.

# 👥 Team

* **Member 1**: [Kavini Pathagamage](https://github.com/kavikushi0207): Infrastructure setup, Dockerization, K8s Orchestration, CI/CD basics, Azure deployment and test bed setup, Experimenting and Analyzing test results

* **Member 2**: [Taniya Afreen](https://github.com/taanyaafreen): Logic/Model development, Model Enhancement, Test dataset preparation, and Performance experiment design, Experimenting and  Analyzing test results
