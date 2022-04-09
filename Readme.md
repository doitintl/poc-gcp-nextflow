#
# In order to set up this demo I followed this tutorial to first try
# to execute a Nextflow job on GCP:
# https://cloud.google.com/life-sciences/docs/tutorials/nextflow
#

#
# First is a good idea to test the behavior of the Life Sciences API executing
# the next command. Notice that I am passing another bucket 'gonzalo-playground-life-sciences':
#

gsutil mb gs://gonzalo-playground-life-sciences

gcloud beta lifesciences pipelines run \
    --regions us-east1 \
    --command-line 'samtools index ${BAM} ${BAI}' \
    --docker-image "gcr.io/cloud-lifesciences/samtools" \
    --inputs BAM=gs://genomics-public-data/NA12878.chr20.sample.bam \
    --outputs BAI=gs://gonzalo-playground-life-sciences/NA12878.chr20.sample.bam.bai



# Complete the following steps to create a service account and add the following Identity and Access Management roles:
# - Create a GCS Bucket:  doit-gonzalo-poc-nextflow
# - Cloud Life Sciences Workflows Runner
# - Service Account User
# - Service Usage Consumer
# - Storage Object Admin


gsutil mb 'gs://doit-gonzalo-poc-nextflow'

export PROJECT=gonzalo-playground
export SERVICE_ACCOUNT_NAME=nextflow-service-account
export SERVICE_ACCOUNT_ADDRESS=${SERVICE_ACCOUNT_NAME}@${PROJECT}.iam.gserviceaccount.com

gcloud iam service-accounts create ${SERVICE_ACCOUNT_NAME}

gcloud projects add-iam-policy-binding ${PROJECT} --member serviceAccount:${SERVICE_ACCOUNT_ADDRESS} \
    --role roles/lifesciences.workflowsRunner

gcloud projects add-iam-policy-binding ${PROJECT}  --member serviceAccount:${SERVICE_ACCOUNT_ADDRESS} \
    --role roles/iam.serviceAccountUser

gcloud projects add-iam-policy-binding ${PROJECT}  --member serviceAccount:${SERVICE_ACCOUNT_ADDRESS} \
    --role roles/serviceusage.serviceUsageConsumer

gcloud projects add-iam-policy-binding ${PROJECT}  --member serviceAccount:${SERVICE_ACCOUNT_ADDRESS} \
    --role roles/storage.objectAdmin

# Provide credentials to your application
# You can provide authentication credentials to your application code or commands by setting 
# the environment variable GOOGLE_APPLICATION_CREDENTIALS to the path of the JSON file that contains your service account key.
# 
export SERVICE_ACCOUNT_KEY=${SERVICE_ACCOUNT_NAME}-private-key.json

gcloud iam service-accounts keys create  --iam-account=${SERVICE_ACCOUNT_ADDRESS} \
  --key-file-type=json ${SERVICE_ACCOUNT_KEY}

export SERVICE_ACCOUNT_KEY_FILE=${PWD}/${SERVICE_ACCOUNT_KEY}
export GOOGLE_APPLICATION_CREDENTIALS=${PWD}/${SERVICE_ACCOUNT_KEY}

# Install and configure Nextflow in Cloud Shell
# 
#export NXF_JAVA_HOME=/usr/local/opt/openjdk@11
export NXF_VER=20.10.0
export NXF_MODE=google
curl https://get.nextflow.io | bash

git clone https://github.com/nextflow-io/rnaseq-nf.git

cd rnaseq-nf
git checkout v2.0

./nextflow run rnaseq-nf/main.nf -profile gls


#
# In order to create a Workflows orchestration pipeline to invoke the Nextflow API
# endpoints (https://tower.nf/openapi/index.html) first will be necessary to create an account
# in Nextflow Tower by Sequera (https://tower.nf/) and setup:
# - setup the organization and workspace
# - setup the computing environment in the Nextflow Tower
# - setup the credentials to invoke GCP service using the compute engine service account
# - setup the token to invoke the Nextflow Tower API
# - test  Launching a pipeline 
# - setup the Action for future launches without the hassle of all the pipeline configuration setup
#

#
# to test the endpoints with the token and obtain the pipelines of the workspace id 271472500620056
#

curl -X GET "https://api.tower.nf/pipelines?workspaceId=271472500620056" \
 -H "Accept: application/json" \
 -H "Authorization: Bearer <TOKEN>" \

# 
# then to submit the execution of a pipeline actionId 'uGqXaBbetUBVISBrIFhzH' from the 
# workspaceId '271472500620056'
# 

curl -X POST "https://api.tower.nf/actions/uGqXaBbetUBVISBrIFhzH/launch?workspaceId=271472500620056" \
   -H "Accept: application/json" \
   -H "Authorization: Bearer <TOKEN>" \
   -H "Content-Type: application/json" \
   -d '{"params":{}}'

#
# After having tested the integration it is possible to create a new Google Workflows pipeline
# using the YAML definitions from the workflows folder
#
# - nextflow-launch.yaml: to invoke a nextflow pipeline
# - poc-nextflow-execution-progress-check.yaml: to check the execution of a pipeline
# - nextflow-all.yaml: to invoke both in a single pipeline
#