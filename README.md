Resume/JD Matcher API: A FaaS Performance Evaluation

# 📌 Project Overview

This project implements a Serverless HTTP API designed to analyze resumes against job descriptions (JD). It utilizes Natural Language Processing (NLP) to return a match score and identifies "skill gaps" to provide career insights. The core objective of this study is to benchmark Function-as-a-Service (FaaS) behaviors, specifically focusing on cold vs. warm start latency, horizontal scaling, and resource management within a Kubernetes environment.

# 🏗 System Architecture

The system follows a containerized microservices pattern:


* Serverless API: Developed using Azure Functions (Python V2).


* NLP Engine: Built with Scikit-learn using TF-IDF Vectorization and Cosine Similarity.


* Containerization: Packaged via Docker (emulated for linux/amd64 compatibility).


* Orchestration: Managed by Kubernetes (Minikube) with automated scaling.


# 🚀 Getting Started

 ## Prerequisites
 
* Docker Desktop

* Minikube & kubectl

* Azure Functions Core Tools

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

#### Installing JMeter on Mac
```
brew install jmeter 
```
#### Open the software
```
jmeter
```
---
### 📊 JMeter Performance Testing Setup

Follow these steps to configure our JMeter test plan for the Azure FaaS application. We will start with a safe baseline test before scaling up to a full stress test.

#### Step 1: Create the "Users" (Thread Group)

A "Thread Group" is JMeter's way of representing a group of users.

1. Look at the left sidebar and **right-click** on the beaker icon named `FaaS testing attempt 1`.
2. Hover over **Add** > **Threads (Users)** > and click **Thread Group**.
3. Click on the new **Thread Group** that appears in the left sidebar.
4. In the main window, set the following values to run a safe baseline test:
   * **Number of Threads (users):** `10`
   * **Ramp-up period (seconds):** `10`
   * **Loop Count:** `1` 
   
   *(Note: This configures JMeter to send 10 users to the API spread evenly over 10 seconds. We will increase this to 500 later for the stress test.)*

---

#### Step 2: Set up the API Call (HTTP Request)

Next, we need to instruct the simulated users on what specific actions to perform.

1. **Right-click** on a new Thread Group in the left sidebar.
2. Hover over **Add** > **Sampler** > and click **HTTP Request**.
3. Click on the new **HTTP Request** in the left sidebar.
4. In the main window, configure the exact details of the request:
   * **Protocol:** `https`
   * **Server Name or IP:** Paste the Azure FQDN here (e.g., `resumematcher-app.proudriver-abcd123.northeurope.azurecontainerapps.io`). *Note: Do NOT include `https://` in this box!*
   * **HTTP Request Method:** Change the dropdown from `GET` to `POST`.
   * **Path:** `/api/match`
5. Look below the Path box and click the **Body Data** tab.
6. Paste this test JSON directly into the empty text area:

```
JSON
{
  "resume": "Software engineer with 5 years of experience in Python, Azure, and Docker. Strong background in FaaS and system reliability.",
  "jd": "Looking for a backend developer skilled in Python, cloud infrastructure, and building scalable API endpoints."
}
```
---
#### Step 3: Add the JSON Header

Because we are sending JSON data, we must explicitly inform Azure of the payload format. Otherwise, your backend Python script will reject the request.

1. **Right-click** on our **HTTP Request** in the left sidebar.
2. Hover over **Add** > **Config Element** > and click **HTTP Header Manager**.
3. Click on the new **HTTP Header Manager** in the left sidebar.
4. At the bottom of the main window, click the **Add** button.
5. In the new row that appears, enter exactly:
   * **Name:** `Content-Type`
   * **Value:** `application/json`

---

#### Step 4: Add the Dashboards (Listeners)

To evaluate the results of the test, we need to set up "Listeners" in JMeter.

1. **Right-click** on the **Thread Group** in the left sidebar.
2. Hover over **Add** > **Listener** > and click **View Results Tree**. 
   *(This allows ou to inspect the exact response Azure returns to verify successful executions versus errors.*
3. **Right-click** our **Thread Group** again.
4. Hover over **Add** > **Listener** > and click **Summary Report**. 
   *(This provides the average response times, throughput, and error percentages needed for our project evaluation).*

---

#### Step 5: Run the Test

1. Click on **View Results Tree** in the left sidebar so we can monitor the requests in real-time.
2. Look at the top toolbar and click the **Green Play Button** (Start).
3. JMeter will prompt the test plan to save  before executing. Save it as `FaaS-Test.jmx` in a preferred directory on the local machine.
---
As part of our research, we are supposed to evaluate the following metrics:


* Cold Start Latency


* Warm Start Latency


* Scalability



# 🛠 Troubleshooting & Insights

* Auth Level: Set to ANONYMOUS for localized benchmarking to eliminate security handshake overhead.

* Resource Constraints: Identified that emulated x86_64 environments on ARM64 hosts require higher CPU/RAM quotas to prevent SIGABRT errors.

# 👥 Team

* **Member 1**: [Kavini Pathagamage](https://github.com/kavikushi0207): Infrastructure setup, Dockerization, K8s Orchestration, CI/CD basics, Azure deployment and test bed setup, Analyzing test results

* **Member 2**: [Taniya Afreen](https://github.com/taanyaafreen): Logic/Model development, Model Enhancement, Test dataset preparation, and Performance experiment design, Analyzing test results
