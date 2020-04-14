# Deploying WordPress on GKE with Persistent Disks and Cloud SQL   

This tutorial shows you how to set up a single-replica WordPress deployment on Google Kubernetes Engine (GKE) using a MySQL database. This solution is backed by PersistentVolumes (PV) and PersistentVolumeClaims (PVC) to store data. WordPress and MySQL deployments use Google Persistent Disk as storage to back the PVs.

As we've discussed pod's root filesystems are ephemeral and are deleted when the pod is deleted or a node fails. 

Using PVs backed by Persistent Disk lets you store your WordPress platform data outside the pods. This way, even if they are deleted, the data is still available.

WordPress requires a PV to store data. For this tutorial, you use the default storage class to dynamically create Google Persistent Disk and create a PVC for the deployment.

You will: 
* Create a PV and a PVC backed by Persistent Disk.
* Deploy MySQL. 
* Deploy WordPress.
* Set up your WordPress blog.

## Creating PV and PVC for Wordpress and MySQL deployments
To create the storage required for WordPress and MySQL, you need to create PVCs. GKE has a default StorageClass resource installed that lets you dynamically provision PVs backed by Persistent Disk. You use the `wordpress-volumeclaim.yaml` and `mysql-volumeclaim.yaml` files to create the PVCs required for the deployments.   

This manifests describe PVCs that request 200 GB of storage. A StorageClass resource hasn't been defined in the file, so these PVCs use the default StorageClass resource to provision PVs backed by Persistent Disks.   
```
kubectl apply -f manifests/wordpress-volumeclaim.yaml   
kubectl apply -f manifests/mysql-volumeclaim.yaml   
```

It can take a little while to provision the PVs and to bind them to your PVC.    

Confirm the PV was created:   
```
kubectl get pv    
```
```
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                           STORAGECLASS   REASON   AGE   
pvc-8641ae8d-7e62-11ea-8514-42010a800058   200Gi      RWO            Delete           Bound    default/mysql-volumeclaim       standard                22m   
pvc-9c3deed9-7e62-11ea-8514-42010a800058   200Gi      RWO            Delete           Bound    default/wordpress-volumeclaim   standard                21m   
```

## Deploying MySQL
Before you can deploy MySQL you need to create a Secret to store the root password.   
```
kubectl create secret generic mysql --from-literal=password=PASSWORD   
```

Now deploy MySQL   
```
kubectl apply -f manifests/mysql.yaml   
```

Check that the pod is running with `kubectl get pods -l app=mysql` (it might take a few minutes):   

You have completed setting up the database for your new WordPress blog.    

## Create MySQL service
Create a service to expose MySQL internally for WordPress.   
```
kubectl create -f mysql-service.yaml
```

## Deploy WordPress:   
Now that MySQL is ready let's deploy WordPress.   

The `wordpress.yaml` manifest file describes a deployment that creates a single pod running a container with a WordPress instance. This container reads the MySQL password from the secret you created.   

Deploy the `wordpress.yaml` manifest file:   
```
kubectl create -f manifests/wordpress.yaml
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

You can also delete the MySQL deployment, re-deploy it and confirm the data has persisted.

## Clean-up    
Delete the Wordpress deployment, Persistent Volume and Persistent Volume Claim   
```
kubectl delete -f manifests/wordpress_cloudsql.yaml
kubectl delete -f manifests/wordpress-volumeclaim.yaml
```
