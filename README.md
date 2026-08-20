# GKE Kubernetes Deployment and Monitoring Lab

A cloud engineering project demonstrating Kubernetes cluster creation, workload deployment, observability setup, troubleshooting, and application redeployment on Google Kubernetes Engine.

This project was completed as part of a Google Cloud hands-on lab environment. It focuses on managing containerized applications on GKE using Kubernetes, Managed Service for Prometheus, Cloud Logging, and Cloud Monitoring.

## Project Type

Cloud Engineering / DevOps / Kubernetes Observability Project

## Overview

In this project, I created a Google Kubernetes Engine cluster, enabled Managed Prometheus, deployed Kubernetes workloads, diagnosed a failed pod deployment caused by an invalid container image, created a logs-based metric and alerting policy, and redeployed the application using a valid image.

The project demonstrates practical skills in Kubernetes operations, cloud monitoring, and troubleshooting cloud-native workloads.

## Technologies Used

- Google Cloud Platform
- Google Kubernetes Engine
- Kubernetes
- Docker
- Artifact Registry
- Cloud Logging
- Cloud Monitoring
- Managed Service for Prometheus
- YAML
- `kubectl`
- `gcloud` CLI

## Architecture

The project includes the following components:

- A GKE cluster with autoscaling enabled
- A Kubernetes namespace for application resources
- A sample Prometheus metrics application
- A `PodMonitoring` resource for Managed Prometheus
- A `helloweb` Kubernetes deployment
- Cloud Logging logs-based metric
- Cloud Monitoring alerting policy
- Application redeployment with a corrected container image

## Project Objectives

The main objectives of this project were to:

1. Create and configure a GKE cluster.
2. Enable Managed Service for Prometheus.
3. Deploy Kubernetes workloads using YAML manifests.
4. Troubleshoot a pod deployment error.
5. Create a logs-based metric for Kubernetes image errors.
6. Configure an alerting policy in Cloud Monitoring.
7. Fix the deployment manifest and redeploy the application successfully.

## Environment Variables

The following environment variables were used during the project:

```bash
export PROJECT_ID="PROJECT_ID"
export CLUSTER_NAME="hello-world-g1la"
export REGION="europe-west4"
export ZONE="europe-west4-c"
export NAMESPACE_NAME="gmp-hp7d"
export REPO_NAME="sandbox-repo"
export SERVICE_NAME="helloweb-service-pa7h"
```

> Note: The project ID has been replaced with a placeholder for privacy.

## Task 1: Create a GKE Cluster

A GKE cluster was created with the following configuration:

| Setting | Value |
|---|---|
| Zone | europe-west4-c |
| Release Channel | Regular |
| Number of Nodes | 3 |
| Autoscaling | Enabled |
| Minimum Nodes | 2 |
| Maximum Nodes | 6 |

Example command:

```bash
gcloud container clusters create $CLUSTER_NAME \
  --zone $ZONE \
  --release-channel regular \
  --num-nodes 3 \
  --enable-autoscaling \
  --min-nodes 2 \
  --max-nodes 6
```

After creating the cluster, credentials were retrieved using:

```bash
gcloud container clusters get-credentials $CLUSTER_NAME \
  --zone $ZONE \
  --project $PROJECT_ID
```

The cluster nodes were verified using:

```bash
kubectl get nodes
```

Expected result:

```text
NAME                                      STATUS   ROLES    AGE   VERSION
gke-hello-world-g1la-default-pool-xxxxx   Ready    <none>   95s   v1.35.x
gke-hello-world-g1la-default-pool-xxxxx   Ready    <none>   95s   v1.35.x
gke-hello-world-g1la-default-pool-xxxxx   Ready    <none>   95s   v1.35.x
```

## Task 2: Enable Managed Prometheus

Managed Service for Prometheus was enabled on the GKE cluster.

```bash
gcloud container clusters update $CLUSTER_NAME \
  --zone $ZONE \
  --enable-managed-prometheus
```

A namespace was created for the project resources:

```bash
kubectl create namespace $NAMESPACE_NAME
```

## Task 3: Deploy a Prometheus Test Application

A sample Prometheus application manifest was downloaded:

```bash
gcloud storage cp gs://spls/gsp510/prometheus-app.yaml .
```

The manifest was updated with the following container configuration:

```yaml
containers:
- image: nilebox/prometheus-example-app:latest
  name: prometheus-test
  ports:
  - name: metrics
    containerPort: 1234
```

The application was deployed into the namespace:

```bash
kubectl apply -f prometheus-app.yaml -n $NAMESPACE_NAME
```

The pods were verified:

```bash
kubectl get pods -n $NAMESPACE_NAME
```

Expected result:

```text
NAME                               READY   STATUS    RESTARTS   AGE
prometheus-test-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
prometheus-test-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
prometheus-test-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
```

## Task 4: Configure Pod Monitoring

A `PodMonitoring` manifest was downloaded:

```bash
gcloud storage cp gs://spls/gsp510/pod-monitoring.yaml .
```

The manifest was updated with the following configuration:

```yaml
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: prometheus-test
  labels:
    app.kubernetes.io/name: prometheus-test
spec:
  selector:
    matchLabels:
      app: prometheus-test
  endpoints:
  - port: metrics
    interval: 50s
```

The `PodMonitoring` resource was applied:

```bash
kubectl apply -f pod-monitoring.yaml -n $NAMESPACE_NAME
```

It was verified using:

```bash
kubectl get podmonitoring -n $NAMESPACE_NAME
```

## Task 5: Deploy an Application with an Invalid Image

The demo application files were downloaded:

```bash
gcloud storage cp -r gs://spls/gsp510/hello-app/ .
```

The `helloweb` deployment manifest was applied:

```bash
kubectl apply -f hello-app/manifests/helloweb-deployment.yaml -n $NAMESPACE_NAME
```

The deployment was checked:

```bash
kubectl get deployments -n $NAMESPACE_NAME
```

The pod status showed an error:

```bash
kubectl get pods -n $NAMESPACE_NAME
```

Observed result:

```text
NAME                         READY   STATUS             RESTARTS   AGE
helloweb-xxxxxxxxxx-xxxxx    0/1     InvalidImageName   0          2m
```

The error was caused by an invalid image reference in the deployment manifest.

Error message:

```text
Failed to apply default image tag "<todo>": couldn't parse image name "<todo>": invalid reference format
Error: InvalidImageName
```

This demonstrated how Kubernetes reports container image issues during deployment.

## Task 6: Create a Logs-Based Metric

Cloud Logging was used to query Kubernetes pod warning logs related to the invalid image error.

Example log query:

```text
resource.type="k8s_pod"
severity=WARNING
(jsonPayload.message:"InvalidImageName" OR jsonPayload.message:"Failed to apply default image tag")
```

A logs-based metric was created with the following name:

```text
pod-image-errors
```

The metric counts Kubernetes pod image errors and can be used to monitor failed deployments caused by invalid container image references.

Example command:

```bash
gcloud logging metrics create pod-image-errors \
  --description="Count Kubernetes pod image errors" \
  --log-filter='resource.type="k8s_pod"
severity=WARNING
(jsonPayload.message:"InvalidImageName" OR jsonPayload.message:"Failed to apply default image tag")'
```

## Task 7: Create an Alerting Policy

A Cloud Monitoring alerting policy was created based on the logs-based metric.

Alert policy configuration:

| Setting | Value |
|---|---|
| Metric | logging.googleapis.com/user/pod-image-errors |
| Rolling Window | 10 minutes |
| Rolling Window Function | Count |
| Time Series Aggregation | Sum |
| Condition Type | Threshold |
| Alert Trigger | Any time series violates |
| Threshold Position | Above threshold |
| Threshold Value | 0 |
| Notification Channel | Disabled |
| Alert Policy Name | Pod Error Alert |

This alert policy helps detect future pod image errors in the Kubernetes cluster.

## Task 8: Fix and Redeploy the Application

The invalid image placeholder in the deployment manifest was replaced with a valid image:

```text
us-docker.pkg.dev/google-samples/containers/gke/hello-app:1.0
```

The manifest was updated using:

```bash
sed -i 's|image: <todo>|image: us-docker.pkg.dev/google-samples/containers/gke/hello-app:1.0|' hello-app/manifests/helloweb-deployment.yaml
```

The existing broken deployment was deleted:

```bash
kubectl delete deployment helloweb -n $NAMESPACE_NAME
```

The corrected deployment manifest was applied:

```bash
kubectl apply -f hello-app/manifests/helloweb-deployment.yaml -n $NAMESPACE_NAME
```

The deployment was verified:

```bash
kubectl get deployments -n $NAMESPACE_NAME
```

Expected result:

```text
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
helloweb          1/1     1            1           8s
prometheus-test   3/3     3            3           60m
```

The pods were also verified:

```bash
kubectl get pods -n $NAMESPACE_NAME
```

Expected result:

```text
NAME                               READY   STATUS    RESTARTS   AGE
helloweb-xxxxxxxxxx-xxxxx          1/1     Running   0          13s
prometheus-test-xxxxxxxxxx-xxxxx   1/1     Running   0          60m
prometheus-test-xxxxxxxxxx-xxxxx   1/1     Running   0          60m
prometheus-test-xxxxxxxxxx-xxxxx   1/1     Running   0          60m
```

The deployment was described to confirm the correct image was being used:

```bash
kubectl describe deployment helloweb -n $NAMESPACE_NAME
```

Confirmed image:

```text
Image: us-docker.pkg.dev/google-samples/containers/gke/hello-app:1.0
```

## Screenshots

### GKE Cluster Created

![GKE Cluster Created](screenshots/cluster-created.png)

### GKE Cluster Overview

![GKE Cluster Overview](screenshots/cluster-overview.png)

### GKE Nodes

![GKE Nodes](screenshots/cluster-nodes.png)

### Invalid Image Error

![Invalid Image Error](screenshots/invalid-image-error.png)

### Application Redeployed Successfully

![Application Redeployed Successfully](screenshots/deployment-fixed.png)

### Running Pods

![Running Pods](screenshots/running-pods.png)

## Key Learnings

Through this project, I practiced:

- Creating and configuring a GKE cluster
- Enabling cluster autoscaling
- Using `kubectl` to manage Kubernetes workloads
- Creating Kubernetes namespaces
- Deploying applications using YAML manifests
- Enabling Managed Service for Prometheus
- Creating `PodMonitoring` resources
- Troubleshooting Kubernetes pod errors
- Investigating `InvalidImageName` deployment failures
- Creating logs-based metrics in Cloud Logging
- Creating alerting policies in Cloud Monitoring
- Updating and redeploying Kubernetes manifests
- Verifying healthy workloads in GKE

## Challenges Faced

One of the main issues encountered was a pod deployment failure caused by an invalid image reference:

```text
InvalidImageName
```

The image field in the Kubernetes manifest contained a placeholder value:

```text
<todo>
```

Kubernetes could not parse this as a valid container image name, which caused the pod to remain in a failed state.

The issue was resolved by replacing the placeholder with a valid container image and redeploying the application.

## Final Result

At the end of the project:

- The GKE cluster was running successfully.
- Managed Prometheus was enabled.
- The Prometheus test application was running.
- Pod monitoring was configured.
- The invalid image error was identified and monitored.
- A logs-based metric and alerting policy were created.
- The `helloweb` application was fixed and redeployed successfully.
- All pods were running without image errors.

## Repository Notes

This repository is intended as a cloud engineering portfolio project. Sensitive information such as project IDs, student accounts, and personal details have been removed or replaced with placeholders.

## Project Category

Cloud Engineering, DevOps, Kubernetes, Observability
