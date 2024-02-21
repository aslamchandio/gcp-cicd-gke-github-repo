# CI/CD Pipeline for Deploying Applications on Google Kubernetes Engine (GKE)
![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images-2/Archtr.jpg)

## Introduction
Overview:

CI/CD is a software development approach that lets you automate the build, test, and deployment phases of software development by using a number of tools and repeatable processes.

In addition to CI/CD automation, Kubernetes and containers have enabled enterprises to achieve unprecedented improvements in the speed of development and deployment. Yet, even as Kubernetes and container adoption grows, many organizations don't fully realize the benefits in release velocity, stability, and operational efficiencies because their CI/CD practices don't take full advantage of Kubernetes or address operations and security concerns.

## Step 1 — Create a repo on Github

gcp-cicd-gke-github-repo (Private)


### References
- https://cloud.google.com/kubernetes-engine/docs/tutorials/modern-cicd-gke-user-guide
- https://cloud.google.com/kubernetes-engine/docs/tutorials/modern-cicd-gke-reference-architecture
- https://medium.com/google-cloud/ci-cd-pipeline-to-deploy-applications-on-google-kubernetes-engine-gke-using-cloud-build-and-cloud-0ee982b37db6


## Step 2 — Code Build yaml file

- cloudbuild.yaml
```
steps:
- name: 'gcr.io/cloud-builders/docker'
  id: 'Build Docker Image App1'
  args:
  - 'build'
  - '-t'
  - 'us-central1-docker.pkg.dev/$PROJECT_ID/gke-cicd-repo/app1'
  - './app1'

  # images:
  # - 'us-central1-docker.pkg.dev/<your_project_id>/gke-repo/quickstart-image'

- name: 'gcr.io/cloud-builders/docker'
  id: 'Push Docker Image App1'
  args:
  - 'push'
  - 'us-central1-docker.pkg.dev/$PROJECT_ID/gke-cicd-repo/app1'

- name: 'gcr.io/cloud-builders/docker'
  id: 'Build Docker Image App2'
  args:
  - 'build'
  - '-t'
  - 'us-central1-docker.pkg.dev/$PROJECT_ID/gke-cicd-repo/app2'
  - './app2'

  # images:
  # - 'us-central1-docker.pkg.dev/<your_project_id>/gke-repo/quickstart-image'

- name: 'gcr.io/cloud-builders/docker'
  id: 'Push Docker Image App2'
  args:
  - 'push'
  - 'us-central1-docker.pkg.dev/$PROJECT_ID/gke-cicd-repo/app2'

- name: 'google/cloud-sdk:latest'
  entrypoint: 'sh'
  args:
  - -xe
  - -c
  - |
    gcloud deploy apply --file deploy/pipeline.yaml --region=us-central1
    gcloud deploy apply --file deploy/dev.yaml --region=us-central1
    gcloud deploy apply --file deploy/prod.yaml --region=us-central1
    gcloud deploy releases create 'app-release-${SHORT_SHA}' --delivery-pipeline=gke-cicd-pipeline --region=us-central1 --skaffold-file=skaffold.yaml


options:
  logging: CLOUD_LOGGING_ONLY

```

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images-2/01-Cloud-Source.jpg)

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images-2/03-Cloud-Build.jpg)


## Step 3 — Create GKE Cluster for Dev & Prod Env

### Step-01: Dev Cluster
- Dev Cluster 
```

gcloud container clusters create dev-cluster1 \
    --region us-central1 \
    --node-locations us-central1-c \
    --cluster-version 1.27.10-gke.1055000 \
    --release-channel "regular" \
    --num-nodes 1 \
    --enable-master-authorized-networks \
    --master-authorized-networks 0.0.0.0/0 \
    --network k8s-vpc \
    --subnetwork k8s-vpc-subnet-us-central1  \
    --cluster-secondary-range-name pod-cidr \
    --services-secondary-range-name service-cidr \
    --enable-private-nodes \
    --enable-ip-alias \
    --master-ipv4-cidr 172.16.1.32/28 \
    --enable-master-global-access \
    --machine-type e2-medium \
    --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver,GcpFilestoreCsiDriver \
    --disk-type pd-balanced  \
    --disk-size 30 \
    --default-max-pods-per-node 110 \
    --enable-dataplane-v2 \
    --enable-autorepair \
    --max-surge-upgrade 1 \
    --max-unavailable-upgrade 0 \
    --no-enable-basic-auth \
    --workload-pool prod-project-424777.svc.id.goog \
    --no-issue-client-certificate

```

 ### Step-02: Prod Cluster
- Dev Cluster 

```

gcloud container clusters create prod-cluster1 \
    --region us-central1 \
    --node-locations us-central1-f \
    --cluster-version 1.27.10-gke.1055000 \
    --release-channel "regular" \
    --num-nodes 1 \
    --enable-master-authorized-networks \
    --master-authorized-networks 0.0.0.0/0 \
    --network k8s-vpc \
    --subnetwork k8s-vpc-subnet-us-central1  \
    --cluster-secondary-range-name pod-cidr \
    --services-secondary-range-name service-cidr \
    --enable-private-nodes \
    --enable-ip-alias \
    --master-ipv4-cidr 172.16.1.0/28 \
    --enable-master-global-access \
    --machine-type e2-medium \
    --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver,GcpFilestoreCsiDriver \
    --disk-type pd-balanced  \
    --disk-size 30 \
    --default-max-pods-per-node 110 \
    --enable-dataplane-v2 \
    --enable-autorepair \
    --max-surge-upgrade 1 \
    --max-unavailable-upgrade 0 \
    --no-enable-basic-auth \
    --workload-pool prod-project-424777.svc.id.goog \
    --no-issue-client-certificate


```

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images-2/02-GKE-Cluster.jpg)


## Step 4 — Create Docker File for App1 & App2
- Docker File for App1

```
FROM nginx
COPY app1-code /usr/share/nginx/html

```
- Docker File for App2

```
FROM nginx
COPY app2-code /usr/share/nginx/html

```

## Step 4 —  Create Cloud Deploy
- pipleline.yaml

```

apiVersion: deploy.cloud.google.com/v1
kind: DeliveryPipeline
metadata:
  name: gke-cicd-pipeline
  labels:
    app: cicd
description: cicd delivery pipeline
serialPipeline:
  stages:
  - targetId: dev
    # profiles:
    # - dev
  # - targetId: staging
  #   profiles:
  #   - staging
  - targetId: prod
  #   profiles:
  #   - prod

```

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images-2/04-Cloud-Deploy1.jpg)


- dev.yaml

```

apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: dev
  annotations: {}
  labels: {}
description: dev
gke:
  cluster: projects/prod-project-424777/locations/us-central1/clusters/dev-cluster1

```

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images-2/04-Cloud-Deploy2.jpg)

- prod.yaml

```
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: prod
  annotations: {}
  labels: {}
description: prod
requireApproval: true
gke:
  cluster: projects/prod-project-424777/locations/us-central1/clusters/prod-cluster1

```

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images-2/04-Cloud-Deploy3.jpg)

## Step 7 —  Use this value as the workload_identity_provider value in the GitHub Actions YAML
- vpc-ci-prod-create.yaml 
- vpc-ci-prod-destroy.yaml 

Note: # Value from command: gcloud iam workload-identity-pools providers describe github-actions --workload-identity-pool="github-actions-pool" --location="global"

```
 workload_identity_provider: "projects/276747595521/locations/global/workloadIdentityPools/github-actions-gke-pool/providers/github-gke-actions"
          create_credentials_file: true
          service_account: "terraform-oidc-gke-sac@terraform-project-335577.iam.gserviceaccount.com"        
          token_format: "access_token"
          access_token_lifetime: "120s"

```

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images-1/gcp-oidc3.jpg)

## Step 8 — Github Actions Files
- vpc-ci-dev-create.yml (For Creating GCP Resources)

 Name: vpc-gke-ci-prod-create
 Branch: GitHub Action Trigger always run from Main branch
 Run Env: Latest Ubuntu with terraform install

 ```
name: vpc-gke-ci-prod-create
run-name: ${{ github.actor }} has triggered the pipeline for Terraform

on:
  push:
    branches:
      - 'main'

defaults:
  run:
    shell: bash
    working-directory: ./gcp-vpc-gke-production
permissions:
  contents: read

jobs:
  deploy-prod:
    runs-on: ubuntu-latest
    # These permissions are needed too interact with GitHub's OIDC Token endpoint. New
    permissions:
      id-token: write 
      contents: read         
    steps:
      - name: "Checkout"
        uses: actions/checkout@v3
      - name: Configure GCP credentials
        id: auth
        uses: google-github-actions/auth@v2
        with:
          # Value from command: gcloud iam workload-identity-pools providers describe github-actions --workload-identity-pool="github-actions-pool" --location="global"
          workload_identity_provider: "projects/276747595521/locations/global/workloadIdentityPools/github-actions-gke-pool/providers/github-gke-actions"
          create_credentials_file: true
          service_account: "terraform-oidc-gke-sac@terraform-project-335577.iam.gserviceaccount.com"        
          token_format: "access_token"
          access_token_lifetime: "120s"
      - name: Echo stuff
        run: printenv
      - name: Setup Terraform with specified version on the runner
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.7.0
          terraform_wrapper: true
             
      # Initialize a new or existing Terraform working directory by creating initial files, loading any remote state, downloading mo
      - name: Terraform init
        id: init
        run: terraform init
      
      # Checks that all Terraform configuration files adhere to a canonical format
      - name: Terraform Format
        id: fmt
        run:  terraform fmt -recursive -write=true 

      # Validate Code
      - name: Terraform validate
        id: validate
        run: terraform validate

      # Generates an execution plan for Terraform
      - name: Terraform plan
        id: plan
        run: terraform plan 
        continue-on-error: true

      - name: Terraform Plan Status
        if: steps.plan.outcome == 'failure'
        run: exit 1

      # On push to "main", build or change infrastructure according to Terraform configuration files
      # Note: It is recommended to set up a required "strict" status check in your repository for "Terraform Cloud". See the documentation on "strict" required status checks for more information: https://help.github.com/en/github/administering-a-repository/types-of-required-status-checks
      - name: Terraform Apply
        run: terraform apply -auto-approve 

```

- vpc-ci-dev-destroy.yml (For Destroying GCP Resources)

Name: vpc-gke-ci-prod-destroy
Branch: GitHub Action Trigger always run from Main branch
Run Env: Latest Ubuntu with terraform install

```
name: vpc-gke-ci-prod-destroy
run-name: ${{ github.actor }} has triggered the pipeline for terraform

on:
  push:
    branches:
      - 'main'

defaults:
  run:
    shell: bash
    working-directory: ./gcp-vpc-gke-production
permissions:
  contents: read
jobs:
  destroy-dev:
    runs-on: ubuntu-latest
    # These permissions are needed too interact with GitHub's OIDC Token endpoint. New
    permissions:
      id-token: write
      contents: read
    steps:
      - name: "Checkout"
        uses: actions/checkout@v4
      - name: Configure GCP credentials
        id: auth
        uses: google-github-actions/auth@v2
        with:
          # Value from command: gcloud iam workload-identity-pools providers describe github-actions --workload-identity-pool="github-actions-pool" --location="global"
          workload_identity_provider: "projects/276747595521/locations/global/workloadIdentityPools/github-actions-gke-pool/providers/github-gke-actions"
          create_credentials_file: true
          service_account: "terraform-oidc-gke-sac@terraform-project-335577.iam.gserviceaccount.com"        
          token_format: "access_token"
          access_token_lifetime: "120s"
      - name: Echo stuff
        run: printenv
      - name: Setup Terraform with specified version on the runner
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: 1.7.0

      # Terraform init    
      - name: Terraform init
        id: init
        run: terraform init
      
      # Destroy All Resources   
      - name: Terraform Destroy
        id : destroy
        run: terraform destroy -auto-approve 

```


- Update provider.tf file for remote terraform.tfstate & locking

```

terraform {
  required_version = "~> 1.7.0" # which means any version equal & above 0.14 like 0.15, 0.16 etc and < 1.xx 
  required_providers {
    google = {
      source = "hashicorp/google"
      #version = "5.13.0"
      version = "~> 5.0"
    }
    kubernetes = {
      source = "hashicorp/kubernetes"
    }

  }

  backend "gcs" {
    bucket = "aslam-terraform-storage"
    prefix = "prod/gke-vpc-module-github/"
  }
}

provider "google" {
  # Configuration options
  project = var.project_id
  region  = var.region
}

```


![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images-1/gcp-oidc4.jpg)

## Step 9 — Create Folders in local Repo in Hierarchical way 

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images-1/gcp-oidc5.jpg)

1- Under aws-oidc-terraform-github repo create folder .github
2- under .github folder create workflows folder
3- under workflows folder create Github action files.

## Step 10 — Create .gitignore file in terraform main folder 

```

.terraform/
.terraform.lock.hcl
**/creds.json
**/zzz.txt
**/.DS_Store

```


## Step 11 —  Push Terraform Code & Github Action files into Remote Repo

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images-1/gcp-oidc6.jpg)

bellow commands used in above images

```

git add .

git status

git commit -m "6th Commit for GKE Code"

git push -u origin main 

```


### References
- https://cloud.google.com/blog/products/identity-security/enabling-keyless-authentication-from-github-actions
- https://github.com/google-github-actions/auth?tab=readme-ov-file#indirect-wif
- https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-google-cloud-platform









































