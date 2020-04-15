# Google Kubernetes Engine Secure Cluster

## Introduction

This lab demonstrates some of the security concerns of a default Kubernetes Engine cluster configuration and the hardening measures to prevent multiple paths of pod escape and cluster privilege escalation.  

The following security settings will be tested in both disabled and enabled states to demonstrate the real-world implications of these configurations:

* Disabling the Legacy GCE Metadata API Endpoint
* Enabling Metadata Concealment
* Enabling and configuring PodSecurityPolicy

### Enable legacy API endpoints 
GKE disables the legacy API in versions 1.12 and newer. To demonstrate how dangerous it can be to leave it enabled we are going to create a cluster that specifically enables it. 
To enable it run the following in Cloud Shell: 

```console
gcloud beta container clusters create deloitte-lab-meta --zone=us-central1-f --metadata=disable-legacy-endpoints=false --num-nodes=1
```


The above command will:

Create a new Kubernetes Engine cluster in your current ZONE, VPC and network that omits configuring the GKE Metadata Concealment proxy and does not enable the setting to block access to the Legacy Compute Metadata API.

### Run a Google Cloud-SDK pod

From your Cloud Shell prompt, launch a single instance of the Google Cloud-SDK container that will be automatically removed after exiting from the shell:

```console
kubectl run -it --generator=run-pod/v1 --rm gcloud --image=google/cloud-sdk:latest --restart=Never -- bash
```

This will take a few moments to complete.

You should now have a bash shell inside the pod's container:

```console
root@gcloud:/#
```

It may take a few seconds for the container to be started and the command prompt to be displayed. If you don't see a command prompt, try pressing __Enter__.

__NOTE: If it hangs for longer than 30 seconds when starting pod delete and re-run above command__.

### Explore the Legacy Compute Metadata Endpoint

In GKE Clusters created with versions 1.11 or below the "Legacy" or `v1beta1` Compute Metadata endpoint is available by default.  Unlike the current Compute Metadata version, `v1`, the `v1beta1` Compute Metadata endpoint does not require a custom HTTP header to be included in all requests.  On new GKE Clusters created at version 1.12 or greater, the legacy Compute Engine metadata endpoints are now disabled by default. 

Run the following command to access the "Legacy" Compute Metadata endpoint without requiring a custom HTTP header to get the GCE Instance name where this pod is running:

```console
curl -s http://metadata.google.internal/computeMetadata/v1beta1/instance/name && echo

gke-default-cluster-default-pool-b57a043a-6z5v
```

The `&& echo` command is to aid with terminal formatting and output readability.  Now, re-run the same command, but instead use the `v1` Compute Metadata endpoint:

```console
curl -s http://metadata.google.internal/computeMetadata/v1/instance/name && echo

...snip...
Your client does not have permission to get URL <code>/computeMetadata/v1/instance/name</code> from this server. Missing Metadata-Flavor:Google header.
...snip...
```

Notice how it returns an error stating that it requires the custom HTTP header to be present.  Add the custom header on the next run and retrieve the GCE instance name that is running this pod:

```console
curl -s -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/name && echo

gke-default-cluster-default-pool-b57a043a-6z5v
```

Without requiring a custom HTTP header when accessing the GCE Instance Metadata endpoint, a flaw in an application that allows an attacker to trick the code into retrieving the contents of an attacker-specified web URL could provide a simple method for enumeration and potential credential exfiltration.  By requiring a custom HTTP header, the attacker needs to exploit an application flaw that allows them to control the URL and also add custom headers in order to carry out this attack successfully.

Keep this shell inside the pod available for the next step.  If you accidentally exit from the pod, simply re-run:

```console
kubectl run -it --generator=run-pod/v1 --rm gcloud --image=google/cloud-sdk:latest --restart=Never -- bash
```

### Explore the GKE node bootstrapping credentials

From inside the same pod shell, run the following command to list the attributes associated with the underlying GCE instances. Be sure to include the trailing slash:

```console
curl -s http://metadata.google.internal/computeMetadata/v1beta1/instance/attributes/
```

Perhaps the most sensitive data in this listing is `kube-env`.  It contains several variables which the `kubelet` uses as initial credentials when attaching the node to the GKE cluster.  The variables `CA_CERT`, `KUBELET_CERT`, and `KUBELET_KEY` contain this information and are therefore considered sensitive to non-cluster administrators.

To see the potentially sensitive variables and data, run the following command:

```console
curl -s http://metadata.google.internal/computeMetadata/v1beta1/instance/attributes/kube-env
```

Therefore, in any of the following situations:

1. A flaw that allows for SSRF in a pod application
2. An application or library flaw that allow for RCE in a pod
3. An internal user with the ability to create or exec into a pod

There exists a high likelihood for compromise and exfiltration of sensitive `kubelet` bootstrapping credentials via the Compute Metadata endpoint.  With the `kubelet` credentials, it is possible to leverage them in certain circumstances to escalate privileges to that of  `cluster-admin` and therefore have full control of the GKE Cluster including all data, applications, and access to the underlying nodes.

### Leverage the Permissions Assigned to this Node Pool's Service Account

By default, GCP projects with the Compute API enabled have a default service account in the format of `NNNNNNNNNN-compute@developer.gserviceaccount.com` in the project and the `Editor` role attached to it.  Also by default, GKE clusters created without specifying a service account will utilize the default Compute service account and attach it to all worker nodes.

Run the following `curl` command to list the OAuth scopes associated with the service account attached to the underlying GCE instance:

```console
curl -s -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/scopes

https://www.googleapis.com/auth/devstorage.read_only
https://www.googleapis.com/auth/logging.write
https://www.googleapis.com/auth/monitoring
https://www.googleapis.com/auth/service.management.readonly
https://www.googleapis.com/auth/servicecontrol
https://www.googleapis.com/auth/trace.append
```

The combination of authentication scopes and the permissions of the service account dictates what applications on this node can access.  The above list is the minimum scopes needed for most GKE clusters, but some use cases require increased scopes.

If the authentication scope were to be configured during cluster creation to include `https://www.googleapis.com/auth/cloud-platform`, this would allow any GCP API to be considered "in scope", and only the IAM permissions assigned to the service account would determine what can be accessed.  If the default service account is in use and the default IAM Role of `Editor` was not modified, this effectively means that any pod on this node pool has `Editor` permissions to the GCP project where the GKE cluster is deployed.  As the `Editor` IAM Role has a wide range of read/write permissions to interact with resources in the project such as Compute instances, GCS buckets, GCR registries, and more, this is most likely not desired.

Exit out of this pod by typing:

```console
exit
```

### Deploy a pod that mounts the host filesystem

One of the simplest paths for "escaping" to the underlying host is by mounting the host's filesystem into the pod's filesystem using standard Kubernetes `volumes` and `volumeMounts` in a `Pod` specification.

To demonstrate this, run the following to create a Pod that mounts the underlying host filesystem `/` at the folder named `/rootfs` inside the container:

```console
kubectl apply -f manifests/hostpath.yml
```

Run `kubectl get pod` and re-run until it's in the "Running" state:

```console
kubectl get pod

NAME       READY   STATUS    RESTARTS   AGE
hostpath   1/1     Running   0          30s
```

### Explore and compromise the underlying host

Run the following to obtain a shell inside the pod you just created:

```console
kubectl exec -it hostpath -- bash
```

Switch to the pod shell's root filesystem point to that of the underlying host:

```console
chroot /rootfs /bin/bash

hostpath / #
```

With those simple commands, the pod is now effectively a `root` shell on the node. You are now able to do the following:

| run the standard docker command with full permissions                          | `docker ps`                                          |
|--------------------------------------------------------------------------------|------------------------------------------------------|
| list all local docker images                                                   | `docker images`                                      |
| `docker run` privileged container of your choosing                             | `docker run --privileged <imagename>:<imageversion>` |
| examine the Kubernetes secrets mounted on the node                             | `mount \| grep volumes \| awk '{print $3}' \| xargs ls` |
| `exec` into any running container (even into another pod in another namespace) | `docker exec -it <docker container ID> sh`           |

Nearly every operation that the `root` user can perform is available to this pod shell.  This includes persistence mechanisms like adding SSH users/keys, running privileged docker containers on the host outside the view of Kubernetes, and much more.

To exit the pod shell, run `exit` twice - once to leave the `chroot` and another to leave the pod's shell:

```console
exit
```

```console
exit
```

Now you can delete the `hostpath` pod:

```console
kubectl delete -f manifests/hostpath.yml

pod "hostpath" deleted
```

### Understand the available controls

The next steps of this demo will cover:

* __Disabling the Legacy GCE Metadata API Endpoint__ - By specifying a custom metadata key and value, the `v1beta1` metadata endpoint will no longer be available from the instance.
* __Enable Metadata Concealment__ - Passing an additional configuration during cluster and/or node pool creation, a lightweight proxy will be installed on each node that proxies all requests to the Metadata API and prevents access to sensitive endpoints.
* __Enable and configure PodSecurityPolicy__ - Configuring this option on a GKE cluster will add the PodSecurityPolicy Admission Controller which can be used to restrict the use of insecure settings during Pod creation.  In this demo's case, preventing containers from running as the root user and having the ability to mount the underlying host filesystem.

### Deploy a second node pool

To enable you to experiment with and without the Metadata endpoint protections in place, you'll create a second node pool that includes two additional settings.  Pods that are scheduled to the generic node pool will not have the protections, and Pods scheduled to the second node pool will have them enabled.

Note: In GKE versions 1.12 and newer, the `--metadata=disable-legacy-endpoints=true` setting will automatically be enabled.  The next command is defining it explicitly for clarity.

Create the second node pool:

```console
gcloud beta container node-pools create second-pool --cluster deloitte-lab --zone us-central1-f --num-nodes=1 --metadata=disable-legacy-endpoints=true --workload-metadata-from-node=SECURE

NAME         MACHINE_TYPE   DISK_SIZE_GB  NODE_VERSION
second-pool  n1-standard-1  100           1.14.10-gke.27
```

### Run a Google Cloud-SDK pod

In Cloud Shell, launch a single instance of the Google Cloud-SDK container that will be run only on the second node pool with the protections enabled and not run as the root user.

```console
kubectl run -it --generator=run-pod/v1 --rm gcloud --image=google/cloud-sdk:latest --restart=Never --overrides='{ "apiVersion": "v1", "spec": { "securityContext": { "runAsUser": 65534, "fsGroup": 65534 }, "nodeSelector": { "cloud.google.com/gke-nodepool": "second-pool" } } }' -- bash
```

You should now have a bash shell inside the pod's container running on the node pool named `second-pool`. You should see the following:

```console
nobody@gcloud:/$
```
It may take a few seconds for the container to be started and the command prompt to be displayed.

If you don't see a command prompt, try pressing __Enter__.

If it appears to hang, delete the pod and run above command to re-create.

### Explore various blocked endpoints

With the configuration of the second node pool set to `--metadata=disable-legacy-endpoints=true`, the following command will now fail as expected:

```console
curl -s http://metadata.google.internal/computeMetadata/v1beta1/instance/name

...snip...
Legacy metadata endpoints are disabled. Please use the /v1/ endpoint.
...snip...
```

With the configuration of the second node pool set to `--workload-metadata-from-node=SECURE` , the following command to retrieve the sensitive file, `kube-env`, will now fail:

```console
curl -s -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/attributes/kube-env

This metadata endpoint is concealed.
```

But other commands to non-sensitive endpoints will still succeed if the proper HTTP header is passed:

```console
curl -s -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/name && echo

gke-default-cluster-second-pool-8fbd68c5-gzzp
```

Exit out of the pod:

```console
exit
```

You should now be back to your shell.

### Deploy PodSecurityPolicy objects

In order to have the necessary permissions to proceed, grant explicit permissions to your own user account to become `cluster-admin:`

```console
kubectl create clusterrolebinding clusteradmin --clusterrole=cluster-admin --user="$(gcloud config list account --format 'value(core.account)')"

clusterrolebinding.rbac.authorization.k8s.io/clusteradmin created
```

Next, deploy a more restrictive `PodSecurityPolicy` on all authenticated users in the default namespace:

```console
kubectl apply -f manifests/restrictive-psp.yml

podsecuritypolicy.extensions/restrictive-psp created
```

Next, add the `ClusterRole` that provides the necessary ability to "use" this PodSecurityPolicy.

```console
kubectl apply -f manifests/restrictive-psp-clusterrole.yml

clusterrole.rbac.authorization.k8s.io/restrictive-psp created
```

Finally, create a RoleBinding in the default namespace that allows any authenticated user permission to leverage the PodSecurityPolicy.

```console
kubectl apply -f manifests/restrictive-psp-clusterrolebinding.yml

rolebinding.rbac.authorization.k8s.io/restrictive-psp created
```

__Note:__ In a real environment, consider replacing the `system:authenticated` user in the ClusterRoleBinding or Namespace RoleBinding with the specific user or service accounts that you want to have the ability to create pods in the default namespace.

### Enable PodSecurity policy

Next, enable the PodSecurityPolicy Admission Controller:

```console
gcloud beta container clusters update deloitte-lab --zone us-central1-f --enable-pod-security-policy
```

This will take a few minutes to complete.

### Deploy a blocked pod that mounts the host filesystem

Because the account used to deploy the GKE cluster was granted cluster-admin permissions in a previous step, it's necessary to create another separate "user" account to interact with the cluster and validate the PodSecurityPolicy enforcement.  To do this, run:

```console
./create-demo-developer.sh -c deloitte-lab

Created service account [demo-developer].
...snip...
Fetching cluster endpoint and auth data.
kubeconfig entry generated for default-cluster.
```

The `create-demo-developer.sh` script will create a new service account named `demo-developer`, grant that service account the `container.developer` IAM role, create a service account key, configure gcloud to use that service account key, and then configure kubectl to use those service account credentials when communicating with the cluster.

Now, try to create another pod that mounts the underlying host filesystem `/` at the folder named `/rootfs` inside the container:

```console
kubectl apply -f manifests/hostpath.yml
```

This output validatates that it's blocked by PSP:

```console
Error from server (Forbidden): error when creating "STDIN": pods "hostpath" is forbidden: unable to validate against any pod security policy: [spec.volumes[0]: Invalid value: "hostPath": hostPath volumes are not allowed to be used]
```

Deploy another pod that meets the criteria of the `restrictive-psp`:

```console
kubectl apply -f manifests/nohostpath.yml

pod/nohostpath created
```

To view the annotation that gets added to the pod indicating which PodSecurityPolicy authorized the creation, run:

```console
kubectl get pod nohostpath -o=jsonpath="{ .metadata.annotations.kubernetes\.io/psp }" && echo

restrictive-psp
```

Congratulations! In this lab you configured a default Kubernetes cluster in Google Kubernetes Engine.  You then probed and exploited the access available to your pod, hardened the cluster, and validated those malicious actions were no longer possible.

## Validation

The following script will validate that the demo is deployed correctly:

```console
./validate.sh -c deloitte-lab
```

## Tear Down

Log back in as your user account.

```console
gcloud auth login
```

The following script will destroy the Kubernetes Engine cluster.

```console
./delete.sh -c deloitte-lab

Fetching cluster endpoint and auth data.
kubeconfig entry generated for default-cluster.
Deleting cluster
Deleting cluster default-cluster...
...snip...
deleted service account [demo-developer@my-project-id.iam.gserviceaccount.com]
```

