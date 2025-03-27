 # Task : Spark ETL Job for Azure and GCP
 
## Project Overview  
This project implements a Spark-based ETL pipeline to process hotel and weather datasets on Azure and GCP.  
 
### Features  
- Ingests hotel and weather data from cloud storage (Azure Blob Storage or Google Cloud Storage).  
- Cleans and enhances data by correcting `Latitude` and `Longitude` values using the OpenCage Geocoding API.  
- Generates a 4-character geohash for valid coordinates to enable location-based processing.  
- Merges hotel and weather datasets through a left join using the generated geohash.  
- Saves the enriched dataset in Parquet format within a Terraform-provisioned Storage Account while maintaining partitioning.  
- Deploys the Spark job on Kubernetes using Terraform scripts and a Docker image tailored to the Spark version.  
- Optimizes resource allocation to efficiently run on free-tier Kubernetes clusters.  
 
 
# 1. Unzipping the Dataset  
The given dataset was provided in a compressed ZIP format. To extract it, we used the following command:  
 
```sh
unzip m06sparkbasics.zip -d /path/to/destination
```
 
After extraction, the dataset was structured as follows:
 
m06sparkbasics/ <br/>
├── hotels.csv <br/>
├── weather/ <br/>
│   ├── year=2017/ <br/>
│   │   ├── month=01/ <br/>
│   │   │   ├── day=01/ <br/>
│   │   │   │   ├── part-00000.parquet <br/>
│   │   │   │   ├── .part-00000.parquet.crc <br/>
│   │   ├── month=02/ <br/>
│   │   │   ├── day=01/ <br/>
│   │   │   │   ├── part-00000.parquet <br/>
│   │   │   │   ├── .part-00000.parquet.crc <br/>
│   ├── year=2016/ <br/>
│   │   ├── month=12/ <br/>
│   │   │   ├── day=31/ <br/>
│   │   │   │   ├── part-00000.parquet <br/>
│   │   │   │   ├── .part-00000.parquet.crc
 
 
# 2. Creating Azure Resources and Uploading Data
 
We provisioned the necessary Azure resources using the following steps:
 
## 2.1 Create a Resource Group
```sh
az group create --name Spark_Basic_Task --location eastus
```
## 2.2 Create a Storage Account
```sh
az storage account create --name jithenda --resource-group Spark_Basic_Task --location eastus --sku Standard_LRS
```
## 2.3 Create a Storage Container
```sh
az storage container create --name tfstate --account-name Spark_Basic_Task
```
## 2.4 Upload Data to Azure Storage
We uploaded the extracted dataset folders (hotels.csv and weather/) to the storage container:
```sh
az storage blob upload --account-name rg-dev-westeurope-wefp --container-name data --name hotels.csv --file /path/to/m06sparkbasics/hotels.csv
```
For recursive upload of the weather data:
```sh
az storage blob upload-batch --account-name spark_sasank --destination sparksasank/weather --source /path/to/m06sparkbasics/weather
```
 
# 3. Execution Results
Below are the execution results for each of the five Python scripts.
 
## part1_read_hotel.py
Description: Reads the hotel dataset and displays the first few rows. <br/><br/>
Execution Output: <br/><br/>
![Hotels Image](outputs/output1.jpg)
 
## part2_read_weather.py
Description: Reads and processes weather data from Parquet files.<br/><br/>
Execution Output:<br/><br/>
![Processed Weather Image](outputs/output2.png)
 
## api_key
![API KEY FROM OPENCAGE](outputs/output3.png)
 
## part3_Left_join.py
Description: Performs a left join between hotels and weather datasets using the geohash key. <br/><br/>
Execution Output:<br/><br/>
Here we can see left join has taken place
![Left Join Image](outputs/output4.png)
<br/><br/>
 
 
# 4. Docker Image Creation
 
To package our Spark ETL job into a Docker image, we follow these steps:
 
## 4.1 Docker Login
 
First, log in to the Docker registry:
```sh
docker login
```
![docker login](outputs/output5.png)
 
## 4.2 Build the Docker Image
 
Run the following command to build the Docker image:
 
```sh
docker build -t my-spark-job:latest -f docker/spark-job.yaml .
```
 Output:
<br/>
![Docker Image](outputs/output6.png)
 
 
# 5 Accessing Azure Kubernetes Service (AKS) and Azure Container Registry (ACR)
 
## 5.1 Retrieve the ACR Login Server
To get the login server URL for your Azure Container Registry, run:
 
```sh
terraform output acr_login_server
```
This command outputs the ACR login name, which can be used to push and pull container images.
![ACR name](outputs/output7.png)
 
## 5.2 Connect to the AKS Cluster
Run the following command to get credentials for the AKS cluster:
 
```sh
az aks get-credentials --resource-group rg-dev-westeurope-pcsd --name aks-dev-westeurope-pcsd
 
This configures kubectl to interact with our AKS cluster.
 
Verify Cluster Nodes
 
To check if the Kubernetes nodes are running, execute:
 
```sh
kubectl get nodes
```
 
## 5.3 List Azure Resource Groups
To see the available resource groups in your Azure subscription, use:
 
```sh
az group list --output table
```
 
![output table](outputs/output8.png)
 
# 6. Working with Docker Images and Azure Container Registry (ACR)
 
## Authenticate with Azure Container Registry
Log in to the target ACR using:
 
```sh
az acr login --name "acrdevwesteurope2ib5.azurecr.io"
```
![ACR login image](outputs/output9.png)
 
## Push the Docker Image to ACR
Once authenticated, push the tagged image to ACR:
 
```sh
docker push acrdevwesteurope2ib5.azurecr.io/spark-python-06:latest
```
![Docker Push](outputs/output10.png)
 
## Set Up and Use a Kubeconfig File
To save the AKS kubeconfig output to a file and set it as the active configuration, run:
 
```sh
terraform output -raw aks_kubeconfig > kubeconfig.yaml
$env:KUBECONFIG="$(Get-Location)/kubeconfig.yaml"
```
This ensures that kubectl uses the correct configuration file to communicate with the AKS cluster.
 
# 7. Deploying Spark Jobs on Kubernetes
 
## 7.1 Verify Kubernetes Cluster Access
 
Before deploying, check if Kubernetes is running:
 
```sh
kubectl get nodes
```
![Image](outputs/image.png)
 
## 7.2 Push the Docker Image
 
Ensure your Spark job Docker image is uploaded to Azure Container Registry (ACR):
 
```sh
docker push acrdevwesteurope2ib5.azurecr.io/spark-python-06:latest
```
 
## 7.3 Create a Kubernetes Service Account
 
Spark jobs require a service account with the necessary permissions. Run:
 
```sh
kubectl auth can-i create pods --namespace=default
kubectl get serviceaccount spark --namespace=default
```
 
If the service account does not exist, create one:
 
```sh
kubectl create serviceaccount spark --namespace=default
```
 
Grant the necessary permissions:
 
```sh
kubectl create clusterrolebinding spark-role --clusterrole=edit --serviceaccount=default:spark --namespace=default
```
 
## 7.4 Create the Spark Job YAML
 
Since Spark on Kubernetes requires job submission via kubectl, create a YAML file.
Create spark-job.yaml inside your docker folder:
 
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: spark-job
  namespace: default
spec:
  template:
    spec:
      serviceAccountName: spark
      restartPolicy: Never
      containers:
      - name: spark-job
        image: acrdevwesteuropepcsd.azurecr.io/spark-python-06:latest
        command: ["/opt/spark/bin/spark-submit"]
        args:
          - "--master"
          - "k8s://https://kubernetes.default.svc:443"
          - "--deploy-mode"
          - "cluster"
          - "--name"
          - "spark-job"
          - "--conf"
          - "spark.executor.instances=2"
          - "--conf"
          - "spark.executor.memory=2g"
          - "--conf"
          - "spark.driver.memory=1g"
          - "--conf"
          - "spark.kubernetes.container.image=acrdevcentralindia4gnf.azurecr.io/spark-python-06:latest"
          - "--conf"
          - "spark.kubernetes.namespace=default"
          - "--conf"
          - "spark.kubernetes.authenticate.driver.serviceAccountName=spark"
          - "--conf"
          - "spark.kubernetes.file.upload.path=file:///opt/src/main/python"
          - "local:///opt/src/main/python/task1.py"
```
 
## 7.5 Deploy the Spark Job to Kubernetes
 
```sh
kubectl apply -f spark-job.yaml
```
![sparksubmit](outputs/output11.png)
 
## 7.6 Monitor the Job Execution
 
```sh
kubectl logs -f <pod-name>
```
![sparksubmit1](outputs/output12.png)
![sparksubmit2](outputs/output13.png)
## Kubernets Status
 
![sparksubmit3](outputs/output14.png)
![sparksubmit4](outputs/output15.png)
 
```sh
kubectl get pods -A
```
![pods](outputs/output16.png)
![pods](outputs/output17.png)
![pods](outputs/output18.png)
 
<br/>
 
## Testing
 
This testing is essential to ensure the reliability, accuracy, and efficiency of data processing operations in Apache Spark.
 
**Note:** The opp.py file contains 'fetch_coords(data, key)', 'create_geohash(lat, lon)'
 
### Testing Phases and Test Cases
 
### 1. Data Validation
- **Test Case:** `test_data_integrity`  
- **Functionality Covered:** Ensures datasets contain expected rows and columns.  
 
### 2. API Interaction
- **Test Case:** `test_fetch_coords`  
- **Functionality Covered:** Mocked API response handling for geolocation.  
 
### 3. Data Transformation
- **Test Case:** `test_create_geohash`  
- **Functionality Covered:** Validates geohash generation from coordinates.  
 
### 4. Data Integration
- **Test Case:** `test_merge_dfs`  
- **Functionality Covered:** Ensures correct merging of hotel and weather data.  
 
## How to Run Tests
1. Install dependencies:  
   ```bash
   pip install pyspark requests pygeohash
 
2. Run the test suite:
  ```bash
  python test.py
  ```
## Output
![OutputTest](outputs/test.png)
 
# 8. Continuous Integration and Continuous Deployment (CI/CD)
 
## 8.1 Manual CI/CD Steps
 
### 8.1.1 Running Tests Locally  
Before deployment, we manually ran unit tests to ensure data integrity and correctness.
 
```sh
pip install -r requirements.txt
python test.py
```
### 8.1.2 Building and Pushing Docker Image
After verifying test results, we built the Docker image and pushed it to Azure Container Registry (ACR).
 
```sh
docker build -t acrdevwesteurope2ib5.azurecr.io/spark-python-06:latest .
docker login acrdevwesteurope2ib5.azurecr.io
docker push acrdevwesteurope2ib5.azurecr.io/spark-python-06:latest
```
 
### 8.1.3 Configuring Kubernetes
To deploy the Spark job on Azure Kubernetes Service (AKS), we first authenticated with the cluster.
 
```sh
az aks get-credentials --resource-group rg-dev-westeurope-2ib5 --name aks-dev-westeurope-2ib5
kubectl get nodes
```
### 8.1.4 Deploying the Spark Job
We applied the Spark job YAML configuration to Kubernetes.
```sh
kubectl apply -f spark-job.yaml
kubectl get pods -A
```
 
### 8.1.5 Monitoring Job Execution
We manually monitored logs to verify successful execution.
```sh
kubectl logs -f <pod-name>
kubectl get jobs
```
![cicd](outputs/cicd.png)
 
## Azure Storage
![resource group](outputs/resource.png)
![resource group1](outputs/resource2.png)
![container](outputs/container.png)
![container1](outputs/container1.png)
![final](outputs/final.png)