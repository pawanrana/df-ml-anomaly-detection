# Building a secure anomaly detection solution using Dataflow, BigQuery ML, and Cloud Data Loss Prevention
This tutorial shows you how to build a secure ML-based network anomaly detection solution for telecommunication networks. This type of solution is used to help identify cybersecurity threats.

This tutorial is intended for data engineers who intends to understand how to approach building and End to End Analytics Solution using Google Cloud Technologies from Data Ingestion (both Batch and Streaming) to Transformation (Aggregations, Feature Extraction, Tokenization using PII data de-identification), Realtime ML based Prediction with focus on Monitoring and Health management capabilities of the End to End Solution.

We also Intend to Demo the ML building capabilities of the BigQuery ML to data scientists and Visualization capabilities of Looker for the data analysts.    

## Reference Architecture (Recreate the diagram below to make to easy to understand)

![ref_arch](diagram/ref_arch.png)

## Table of Contents  
* [Anomaly detection in Netflow log](#anomaly-detection-in-netflow-log).  
	* [Initial One time Setup] (#initial-setup)
  * [Raw Data Ingestion](#anomaly-detection-reference-architecture-using-bqml).      
	* [Data Transformation](#quick-start).   
	* [Train & Normalize Data Using BQ ML](#create-a-k-means-model-using-bq-ml )
	* [Feature Extraction Using Dataflow](#feature-extraction-after-aggregation). 
	* [Realtime outlier detection using Dataflow](#find-the-outliers). 
	* [Sensitive data (IMSI) de-identification using Cloud DLP](#dlp-integration). 
	* [Looker Integration](#looker-integration). 
	

## Initial One time Setup
1. In the Google Cloud Console, on the project selector page, select or create a Google Cloud project.

Note: If you don't plan to keep the resources that you create in this procedure, create a project instead of selecting an existing project. After you finish these steps, you can delete the project, removing all resources associated with the project.
[Go to project selector] (https://console.cloud.google.com/projectselector2/home/dashboard)

2. Make sure that billing is enabled for your Cloud project. [Learn how to confirm that billing is enabled for your project.] (https://cloud.google.com/billing/docs/how-to/modify-project)

3. In the Cloud Console, activate Cloud Shell.

[Activate Cloud Shell] (https://console.cloud.google.com/?cloudshell=true)

At the bottom of the Cloud Console, a Cloud Shell session starts and displays a command-line prompt. Cloud Shell is a shell environment with the Cloud SDK already installed, including the gcloud command-line tool, and with values already set for your current project. It can take a few seconds for the session to initialize.

You run all commands in this guide from the Cloud Shell.

4. In Cloud Shell, enable the BigQuery, Dataflow, Cloud Storage, and DLP APIs.

```
gcloud services enable dlp.googleapis.com bigquery.googleapis.com \
  dataflow.googleapis.com storage-component.googleapis.com \
  pubsub.googleapis.com cloudbuild.googleapis.com
  ```

5. Run following commands in Cloud Shell to create a Pub/Sub topic and a subscription

```
export PROJECT_ID=$(gcloud config get-value project)
export TOPIC_ID=demo-anomaly-detect
export SUBSCRIPTION_ID=demo-anomaly-detect-sub
export REGION=us-central1
gcloud pubsub topics create $TOPIC_ID
gcloud pubsub subscriptions create $SUBSCRIPTION_ID --topic=$TOPIC_ID 
```

6. Run following commands in Cloud Shell to clone the GitHub repository:

```
git clone https://github.com/GoogleCloudPlatform/df-ml-anomaly-detection.git
cd df-ml-anomaly-detection
```

7. For Cloud Build To enable submitting a job automatically, grant Dataflow permissions to your Cloud Build service account:

```
export PROJECT_NUMBER=$(gcloud projects list --filter=${PROJECT_ID} \
  --format="value(PROJECT_NUMBER)")

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com \
  --role roles/dataflow.admin

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com \
  --role roles/compute.instanceAdmin.v1

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com \
  --role roles/iam.serviceAccountUser
```

8. In Cloud Shell, create a BigQuery dataset and necessary Tables

```
export DATASET_NAME=demoanomalydetect1
bq --location=US mk -d \
  --description "Network Logs Dataset" \
  ${DATASET_NAME}

bq mk -t --schema src/main/resources/netflow_log_raw_data.json \
  --time_partitioning_type=DAY \
  --clustering_fields="geoCountry,geoCity" \
  --description "Raw Netflow Log Data" \
  ${PROJECT_ID}:${DATASET_NAME}.netflow_log_data

bq mk -t --schema src/main/resources/aggr_log_table_schema.json \
  --time_partitioning_type=DAY \
  --clustering_fields="dst_subnet,subscriber_id" \
  --description "Network Log Feature Table" \
  ${PROJECT_ID}:${DATASET_NAME}.cluster_model_data

bq mk -t --schema src/main/resources/outlier_table_schema.json \
  --description "Network Log Outlier Table" \
  ${PROJECT_ID}:${DATASET_NAME}.outlier_data

bq mk -t --schema src/main/resources/normalized_centroid_data_schema.json \
  --description "Sample Normalized Data" \
  ${PROJECT_ID}:${DATASET_NAME}.normalized_centroid_data
```
The following tables are generated:

netflow_log_data: a clustered partition table that stores the raw netflow log data as ingested from source
cluster_model_data: a clustered partition table that stores feature values for model creation.
outlier_data: an outlier table that stores anomalies.
normalized_centroid_data: a table pre-populated with normalized data created from a sample model.

9. Load the Centroid Sample data into Centroid Table

```
bq load \
  --source_format=NEWLINE_DELIMITED_JSON \
  ${PROJECT_ID}:${DATASET_NAME}.normalized_centroid_data \
  gs://df-ml-anomaly-detection-mock-data/sample_model/normalized_centroid_data.json src/main/resources/normalized_centroid_data_schema.json
```

10. In Cloud Shell, create a Docker image in your project:

```
gcloud auth configure-docker
gradle jib --image=gcr.io/${PROJECT_ID}/df-ml-anomaly-detection:latest -DmainClass=com.google.solutions.df.log.aggregations.SecureLogAggregationPipeline
```

11. Upload the Flex Template configuration file to the Cloud Storage bucket that you created earlier:

```
export DF_TEMPLATE_CONFIG_BUCKET=${PROJECT_ID}-anomaly-config
gsutil mb -c standard -l ${REGION} gs://${DF_TEMPLATE_CONFIG_BUCKET}
cat << EOF | gsutil cp - gs://${DF_TEMPLATE_CONFIG_BUCKET}/dynamic_template_secure_log_aggr_template.json
{"image": "gcr.io/${PROJECT_ID}/df-ml-anomaly-detection",
"sdk_info": {"language": "JAVA"}
}
EOF
```

12. Create a SQL file to pass the normalized model data as a pipeline parameter:

```
echo "SELECT * FROM \`${PROJECT_ID}.${DATASET_NAME}.normalized_centroid_data\`" > normalized_cluster_data.sql
gsutil cp normalized_cluster_data.sql gs://${DF_TEMPLATE_CONFIG_BUCKET}/
```

13. Create the end to end anomaly detection pipeline:

```
gcloud beta dataflow flex-template run "anomaly-detection-with-dlp" \
--project=${PROJECT_ID} \
--region=${REGION} \
--template-file-gcs-location=gs://${DF_TEMPLATE_CONFIG_BUCKET}/dynamic_template_secure_log_aggr_template.json \
--parameters=autoscalingAlgorithm="NONE",\
numWorkers=5,\
maxNumWorkers=5,\
workerMachineType=n1-highmem-4,\
subscriberId=projects/${PROJECT_ID}/subscriptions/${SUBSCRIPTION_ID},\
tableSpec=${PROJECT_ID}:${DATASET_NAME}.cluster_model_data,\
batchFrequency=2,\
customGcsTempLocation=gs://${DF_TEMPLATE_CONFIG_BUCKET}/temp,\
tempLocation=gs://${DF_TEMPLATE_CONFIG_BUCKET}/temp,\
clusterQuery=gs://${DF_TEMPLATE_CONFIG_BUCKET}/normalized_cluster_data.sql,\
outlierTableSpec=${PROJECT_ID}:${DATASET_NAME}.outlier_data,\
inputFilePattern=gs://df-ml-anomaly-detection-mock-data/flow_log*.json,\
workerDiskType=compute.googleapis.com/projects/${PROJECT_ID}/zones/us-central1-b/diskTypes/pd-ssd,\
diskSizeGb=5,\
windowInterval=10,\
writeMethod=FILE_LOADS,\
streaming=true,\
logTableSpec=${PROJECT_ID}:${DATASET_NAME}.netflow_log_data
```

In the Cloud Console, go to the Dataflow page.

Go to the Dataflow page

Click the netflow-anomaly-detection-date +%Y%m%d-%H%M%S-%N` job. A representation of the Dataflow pipeline that's similar to the following appears:

## Raw Data Ingestion
This section of the repo contains a reference implementation of an ML based Network Anomaly Detection solution by using Pub/Sub, Dataflow, BQML & Cloud DLP.  It uses an easy to use built in K-Means clustering model as part of BQML to train and normalize netflow log data.   Key part of the  implementation  uses  Dataflow for  feature extraction & real time outlier detection which  has been tested to process over 20TB of data. (250k msg/sec). Finally, it also uses Cloud DLP to tokenize IMSI  (international mobile subscriber identity) number as the streaming Dataflow pipeline  ingests millions of netflow log form Pub/Sub.   

Securing its internal network from malware and security threats is critical at many customers. With the ever changing malware landscape and explosion of activities in IoT and M2M, existing signature based solutions for malware detection are no longer sufficient. This PoC highlights an ML based network anomaly detection solution using PubSub, Dataflow, BQ ML and DLP to detect mobile malware on subscriber devices and suspicious behaviour in wireless networks.

This solution implements the reference architecture highlighted below. You will execute a <b>dataflow streaming pipeline</b> to process netflow log from GCS and/or PubSub to find outliers in netflow logs  in real time.  This solution also uses a built in K-Means Clustering Model created by using <b>BQ-ML</b>. To see a step-by-step tutorial that walks you through implementing this solution, see [Building a secure anomaly detection solution using Dataflow, BigQuery ML, and Cloud Data Loss Prevention](https://cloud.google.com/solutions/building-anomaly-detection-dataflow-bigqueryml-dlp).   

In summary, you can use this solution to demo following 3 use cases :

1.  Streaming Analytics at Scale by using Dataflow/Beam. (Feature Extraction & Online Prediction).  
2. Making Machine Learning easy to do by creating a model by using BQ ML K-Means Clustering.  
3. Protecting sensitive information e.g:"IMSI (international mobile subscriber identity)" by using Cloud DLP crypto based tokenization.  

## Anomaly Detection Reference Architecture Using BQML


![ref_arch](diagram/ref_arch.png)

## Quick Start

[![Open in Cloud Shell](http://gstatic.com/cloudssh/images/open-btn.svg)](https://console.cloud.google.com/cloudshell/editor?cloudshell_git_repo=https://github.com/GoogleCloudPlatform/df-ml-anomaly-detection.git)

### Enable APIs

```gcloud services enable bigquery
gcloud services enable storage_component
gcloud services enable dataflow
gcloud services enable cloudbuild.googleapis.com
gcloud config set project <project_id>
```
### Access to Cloud Build Service Account 

```export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects list --filter=${PROJECT_ID} --format="value(PROJECT_NUMBER)") 
gcloud projects add-iam-policy-binding ${PROJECT_ID} --member serviceAccount:$PROJECT_NUMBER@cloudbuild.gserviceaccount.com --role roles/editor
gcloud projects add-iam-policy-binding ${PROJECT_ID} --member serviceAccount:$PROJECT_NUMBER@cloudbuild.gserviceaccount.com --role roles/storage.objectAdmin
```

#### Export Required Parameters 
```
export DATASET=<var>bq-dataset-name</var>
export SUBSCRIPTION_ID=<var>subscription_id</var>
export TOPIC_ID=<var>topic_id</var>
export DATA_STORAGE_BUCKET=${PROJECT_ID}-<var>data-storage-bucket</var>
```
You can also export DLP template and batch size to enable DLP transformation in the pipeline
* Batch Size is in bytes and max allowed is less than 520KB/payload
```
export DEID_TEMPLATE=projects/{id}/deidentifyTemplates/{template_id}
export BATCH_SIZE = 350000
```
#### Trigger Cloud Build Script

```
gcloud builds submit scripts/. --config scripts/cloud-build-demo.yaml  --substitutions \
_DATASET=$DATASET,\
_DATA_STORAGE_BUCKET=$DATA_STORAGE_BUCKET,\
_SUBSCRIPTION_ID=${SUBSCRIPTION_ID},\
_TOPIC_ID=${TOPIC_ID},\
_API_KEY=$(gcloud auth print-access-token)
```
#### (Optional) Trigger the pipelines using flex template
If you have all other resources like BigQuery tables, PubSub topic and subscriber, GCS bucket already exist or created before, you can use the command below to trigger the pipeline by using a public image. This may be helpful for run the pipeline for live demo. 

Generate 10k msg/sec of random net flow log data:
```
gcloud beta dataflow flex-template run data-generator --project=<project_id> --region=<region> --template-file-gcs-location=gs://df-ml-anomaly-detection-mock-data/dataflow-flex-template/dynamic_template_data_generator_template.json --parameters=autoscalingAlgorithm="NONE",numWorkers=5,maxNumWorkers=5,workerMachineType=n1-standard-4,qps=10000,schemaLocation=gs://df-ml-anomaly-detection-mock-data/schema/next-demo-schema.json,eventType=net-flow-log,topic=projects/<project_id>/topics/events
```
Generate 1k msg/sec of random outlier data:
```
gcloud beta dataflow flex-template run data-generator --project=<project_id> --region=<region> --template-file-gcs-location=gs://df-ml-anomaly-detection-mock-data/dataflow-flex-template/dynamic_template_data_generator_template.json --parameters=autoscalingAlgorithm="NONE",numWorkers=5,maxNumWorkers=5,workerMachineType=n1-standard-4,qps=1000,schemaLocation=gs://df-ml-anomaly-detection-mock-data/schema/next-demo-schema-outlier.json,eventType=net-flow-log,topic=projects/<project_id>/topics/<topic_id>
```

