# CI/CD Pipeline for Deploying Applications on Google Kubernetes Engine (GKE)
![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images-2/Archtr.jpg)

## Introduction
Overview:

CI/CD is a software development approach that lets you automate the build, test, and deployment phases of software development by using a number of tools and repeatable processes.

In addition to CI/CD automation, Kubernetes and containers have enabled enterprises to achieve unprecedented improvements in the speed of development and deployment. Yet, even as Kubernetes and container adoption grows, many organizations don't fully realize the benefits in release velocity, stability, and operational efficiencies because their CI/CD practices don't take full advantage of Kubernetes or address operations and security concerns.

## Step 1 — Create a repo on Github

gcp-cicd-gke-github-repo (Private)

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

## Step 5 —  Create Cloud Deploy
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

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images-2/05-Cloud-Deploy2.jpg)

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

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images-2/06-Cloud-Deploy3.jpg)

## Step 6 —  Create Skaffold file for Code Deploy

- skaffold.yaml

```
apiVersion: skaffold/v4beta9
kind: Config
build:
  tagPolicy:
    gitCommit: {}
  local: {}
manifests:
  rawYaml:
    - ./kubernetes/*
deploy:
  kubectl: {}
  logs:
    prefix: container

```

## Step 7 —  Deploy Application on Both Clusters - Kubernetes Folder

- app1.yaml

```

apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app1-deployment
  labels:
    app: my-app1-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app1

  template:
    metadata:
      name: my-app1
      labels:
        app: my-app1
    spec:
      containers:
        - name: my-container
          image: us-central1-docker.pkg.dev/prod-project-424777/gke-cicd-repo/app1
          ports:
            - containerPort: 80
---            

apiVersion: v1 
kind: Service
metadata: 
  name: my-app1-lb-service
  labels:
    app: my-app1-lb-service

spec:
  type: LoadBalancer
  selector:
    app: my-app1
  ports:
    - name: http
      port: 80
      targetPort: 80  


```

- app2.yaml

```

apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app2-deployment
  labels:
    app: my-app2-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app2

  template:
    metadata:
      name: my-app2
      labels:
        app: my-app2
    spec:
      containers:
        - name: my-container
          image: us-central1-docker.pkg.dev/prod-project-424777/gke-cicd-repo/app2
          ports:
            - containerPort: 80
---            

apiVersion: v1 
kind: Service
metadata: 
  name: my-app2-lb-service
  labels:
    app: my-app2-lb-service

spec:
  type: LoadBalancer
  selector:
    app: my-app2
  ports:
    - name: http
      port: 80
      targetPort: 80 

```

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images-2/07-GKE-Cluster1.jpg)


## Step 8 —  Artifact Registry

- artifact registry docker images

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images-2/08-Artifact.jpg)

## Step 9 — Application 

- Application Output on Dev Cluster

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images-2/09-App1.jpg)

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images-2/10-App2.jpg)

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images-2/11-App3.jpg)

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images-2/12-gke-dev.jpg)


- Application Output on Prod Cluster

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images-2/13-App1.jpg)

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images-2/14-App2.jpg)

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images-2/15-App3.jpg)

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images-2/16-gke-prod.jpg)



### References
- https://cloud.google.com/kubernetes-engine/docs/tutorials/modern-cicd-gke-user-guide
- https://cloud.google.com/kubernetes-engine/docs/tutorials/modern-cicd-gke-reference-architecture
- https://medium.com/google-cloud/ci-cd-pipeline-to-deploy-applications-on-google-kubernetes-engine-gke-using-cloud-build-and-cloud-0ee982b37db6
























































