# Autoscaling lab (VPA & CA)

In this lab you will enable Cluster Autoscaler and Vertical Pod Autoscaler. These can be used to dynamically 
adjust a deployments requested memory and CPU as well as automatically provision new nodes if required. 

## Update cluster to support vertical pod autoscaler.    
```
gcloud container clusters update deloitte-lab --enable-vertical-pod-autoscaling --zone us-central1-f
```

After the cluster has completed updating run the following to enable node autoprovisioning (Cluster Autoscaler).   
```
gcloud container clusters update deloitte-lab --enable-autoprovisioning --max-cpu 50 --max-memory 1000 --zone us-central1-f
```

In some ways, the Kubernetes CPU and RAM allocation model is a bit of a trap: Request too much and the underlying cluster is less efficient; request too little and you put the entire service at risk. 
Combining Vertical Pod Autoscaler and Cluster Autoscaler dynamically updates resource limits and avoids complex performance testing.

Vertical Pod Autoscaler is inspired by a Google Borg service called AutoPilot. It does three things:

1. It observes the service’s resource utilization for the deployment.

2. It recommends resource requests.

3. It automatically updates the pods’ resource requests, both for new pods as well as for current running pods.

## Deploy a test application that requires a large amount of CPU. 
```
kubectl apply -f manifests/vpa-dep.yaml 
```

Confirm Pods are running. You should see something similar to: 
```
NAME                      READY   STATUS    RESTARTS   AGE
stress-6f489958c8-g97bs   1/1     Running   0          106s
stress-6f489958c8-pz26v   1/1     Running   0          106s
```

Check the resources being consumed: 
```
kubectl top pods   

NAME                      CPU(cores)   MEMORY(bytes)   
stress-6f489958c8-g97bs   950m         1Mi             
stress-6f489958c8-pz26v   958m         1Mi            
```

We can see that the  application is using the entire CPU of available on each VM.   
Now deploy the Vertical Pod Autoscaler with automatic updates disabled to determine what it recommends. 
```
kubectl apply -f manifests/vpa.yaml
```

After a couple mintues check recommendations:   

```
kubectl describe vpa
```

Read through the output paying attention to the `Recommendation` section, which will look similar to: 
```
Recommendation:
    Container Recommendations:
      Container Name:  cpu-demo
      Lower Bound:
        Cpu:     926m
        Memory:  262144k
      Target:
        Cpu:     1101m
        Memory:  262144k
      Uncapped Target:
        Cpu:     1101m
        Memory:  262144k
      Upper Bound:
        Cpu:     100191m
        Memory:  1046500k
```

After observing the workload for a short time, Vertical Pod Autoscaler provides some initial low-confidence recommendations 
for adjusting the pod spec, including the target as well as upper and lower bounds.   

Now change `vpa.yaml` so that `updateMode` is set to `Auto` as below:  
```
updatePolicy:
    updateMode: "Auto"
```

Apply the changed file: 
```
kubectl apply -f manifests/vpa.yaml
```

Look at the Pods CPU requests:
```
kubectl get pod  -o=custom-columns=NAME:.metadata.name,PHASE:.status.phase,CPU-REQUEST:.spec.containers\[0\].resources.requests.cpu
```

This shows that the Pods are requesting more resources than are currently available in our cluster. 

After waiting up to 10 minutes you will see new nodes being provisioned to handle the Pod requests. 

Run the following command to see the current `cluster-autoscaler` status. 

```
kubectl get configmap cluster-autoscaler-status -n kube-system -o yaml
```

## Scale down 
Now if you delete the deployment you should see the nodes being removed after 5 - 10 minutes because they are no longer needed.

## Cleanup
This lab requires additional resources so please remember to clean it up when done. 
```
kubectl delete -f  manifests/
```

## Congrats! You successfully updated the cluster, deployed VPA and configured autoscaling of the nodes to handle resource requests.
