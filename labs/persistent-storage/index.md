# Deploying WordPress on GKE with Persistent Disks and Cloud SQL   

This tutorial shows you how to set up a single-replica WordPress deployment on Google Kubernetes Engine (GKE) using a MySQL database. In this lab you will use Cloud SQL, which provides a managed version of MySQL. WordPress uses PersistentVolumes (PV) and PersistentVolumeClaims (PVC) to store data. WordPress uses Google Persistent Disk as storage to back the PVs.

As we've discussed pod's root filesystems are ephemeral and are deleted when the pod is deleted or a node fails. 

Using PVs backed by Persistent Disk lets you store your WordPress platform data outside the pods. This way, even if they are deleted, the data is still available.

WordPress requires a PV to store data. For this tutorial, you use the default storage class to dynamically create Google Persistent Disk and create a PVC for the deployment.

You will: 
* Create a PV and a PVC backed by Persistent Disk.
* Create a Cloud SQL for MySQL instance.
* Deploy WordPress.
* Set up your WordPress blog.

Start by enabling the Cloud SQL Admin API:
```
gcloud services enable sqladmin.googleapis.com
```
**NOTE:** This may take a few minutes to complete: 

## Creating a PV and a PVC
To create the storage required for WordPress, you need to create a PVC. GKE has a default StorageClass resource installed that lets you dynamically provision PVs backed by Persistent Disk. You use the `wordpress-volumeclaim.yaml` file to create the PVCs required for the deployment.   

This manifest file describes a PVC that requests 200 GB of storage. A StorageClass resource hasn't been defined in the file, so this PVC uses the default StorageClass resource to provision a PV backed by Persistent Disk.
```
kubectl apply -f manifests/wordpress-volumeclaim.yaml   
```

It can take a little while to provision the PV and to bind it to your PVC.   

Confirm the PV was created:   
```
kubectl get pv 
```
```
NAME                    STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
wordpress-volumeclaim   Bound     pvc-89d49350-3c44-11e8-80a6-42010a800002   200G       RWO            standard       5s
```

## Creating a Cloud SQL for MySQL instance
Create an instance named mysql-wordpress-instance:   
```
INSTANCE_NAME=mysql-wordpress-instance
gcloud sql instances create $INSTANCE_NAME
```
Add the instance connection name as an environment variable:   
```
export INSTANCE_CONNECTION_NAME=$(gcloud sql instances describe $INSTANCE_NAME --format='value(connectionName)')   
```

Create a database user called wordpress and a password for WordPress to authenticate to the instance:   
```
CLOUD_SQL_PASSWORD=$(openssl rand -base64 18)
gcloud sql users create wordpress --host=% --instance $INSTANCE_NAME --password $CLOUD_SQL_PASSWORD
```

If you close your Cloud Shell session, you lose the password. Make a note of the password because you need it later in the lab.   

You have completed setting up the database for your new WordPress blog.   

## Deploying Wordpress   
Before you can deploy WordPress, you need to create a service account. You create a Kubernetes secret to hold the service account credentials and another secret to hold the database credentials.   

### Configure a service account and create secrets

To allow your WordPress app to access the MySQL instance through a Cloud SQL proxy, create a service account:
```
SA_NAME=cloudsql-proxy
gcloud iam service-accounts create $SA_NAME --display-name $SA_NAME
```

Add the service account email address as an environment variable:
```
SA_EMAIL=$(gcloud iam service-accounts list \
    --filter=displayName:$SA_NAME \
    --format='value(email)')
```

Add the cloudsql.client role to your service account:   
```
gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID \
    --role roles/cloudsql.client \
    --member serviceAccount:$SA_EMAIL
```

Create a key for the service account:   
```
gcloud iam service-accounts keys create manifests/key.json \
    --iam-account $SA_EMAIL
```
This command downloads a copy of the key.json file.   

Create a secret for the MySQL credentials:   
```
kubectl create secret generic cloudsql-db-credentials \
    --from-literal username=wordpress \
    --from-literal password=$CLOUD_SQL_PASSWORD
```

Create a secret for the service account credentials:   
```
kubectl create secret generic cloudsql-instance-credentials \
    --from-file manifests/key.json
```

### Deploy WordPress:   
Now that all the pre-reqs are done let's deploy WordPress.   

The `wordpress_cloudsql.yaml` manifest file describes a deployment that creates a single pod running a container with a WordPress instance. This container reads the `WORDPRESS_DB_PASSWORD` environment variable that contains the `cloudsql-db-credentials` secret you created.

This manifest file also configures the WordPress container to communicate with MySQL through the Cloud SQL proxy running in the sidecar container. The host address value is set on the `WORDPRESS_DB_HOST` environment variable.

Prepare the deployment file by replacing environment variables:   
```
cat manifests/wordpress_cloudsql.yaml.template | envsubst > \
    manifests/wordpress_cloudsql.yaml
```

Deploy the wordpress_cloudsql.yaml manifest file:   
```
kubectl create -f manifests/wordpress_cloudsql.yaml
```
It takes a few minutes to deploy this manifest file while Persistent Disk are attached to the compute node.   

Watch the deployment and confirm it goes into a `Running` status:

```
kubectl get pod -l app=wordpress --watch
```

### Expose the WordPress service  
In the previous step, you deployed a WordPress container, but it's currently not accessible from outside your cluster because it doesn't have an external IP address.   

Create a service of `type:LoadBalancer`:    

```
kubectl create -f manifests/wordpress-service.yaml 

```
It takes a few minutes to create a load balancer.   

Watch the deployment and wait for the service to have an external IP address assigned:  

## Setting up your WordPress blog
In this section, you set up your WordPress blog.  

In your browser go to the `EXTERNAL-IP`

Run through the WordPress installation, providing the requested information and then login with the credentials you created previously. 


## Confirm persistent disk is maintaining data. 

Now delete the WordPress deployment, and then re-deploy it and confirm the PV has persisted. 
