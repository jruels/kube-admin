# GCP Console
## Log into Google Cloud Portal
1. In a browser open [GCP Portal](https://console.cloud.google.com)
2. Log in using credentials provided in VM spreadsheet.   

## Spin up a Kubernetes cluster
In this section, you will be using Cloud Shell to create a Kubernetes cluster.
1. Click the shell icon on the top right of the browser.
2. In the new console set the project (replacing with actual project id)

```
gcloud config set project <YOUR_PROJECT_ID>
```

3. In the new shell enable Kubernetes Engine API
```
gcloud services enable container.googleapis.com
```

4. Spin up Kubernetes cluster
```
gcloud container clusters create <CLUSTER_NAME> \
    --zone=us-central1-f \
    --machine-type=n1-standard-2 \
    --num-nodes=3
```

After this command completes you can confirm the cluster is running:
```
kubectl get pods --all-namespaces
```

## Lab Complete!
