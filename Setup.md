# Realtime Anomaly Detection Using Google Cloud Stream Analytics and AI Services
This repo provides a reference implementation of a Cloud Dataflow streaming pipelines that integrates with BigQuery ML, Cloud AI Platform, and AutoML (coming soon!) to perform anomaly detection use case as part of real time AI pattern. It contains reference implementations for the following real time anomaly detection use cases:  

1. Finding anomalous behaviour in netflow log to identify cyber security threat for a Telco use case.
2. Finding anomalous transaction to identify fraudulent activities for a Financial Service use case.     

## Table of Contents  
* [Anomaly detection in Netflow log](#anomaly-detection-in-netflow-log).  
	* [Reference Architecture](#anomaly-detection-reference-architecture-using-bqml).      
	* [Quick Start](#quick-start).  
	* [Learn More](#learn-more-about-this-solution). 
	* [Mock Data Generator to Pub/Sub Using Dataflow](#test). 
	* [Train & Normalize Data Using BQ ML](#create-a-k-means-model-using-bq-ml )
	* [Feature Extraction Using Dataflow](#feature-extraction-after-aggregation). 
	* [Realtime outlier detection using Dataflow](#find-the-outliers). 
	* [Sensitive data (IMSI) de-identification using Cloud DLP](#dlp-integration). 
	* [Looker Integration](#looker-integration). 
	
* [Anomaly detection in Financial Transactions](#anomaly-detection-in-financial-transactions). 
	* [Reference Architecture](#anomaly-detection-reference-architecture-using-cloud-ai). 
	* [Build & Run](#build--run-1)
	* [Test](#test-1)
	
	  		 
## Anomaly Detection in Netflow log
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
test
