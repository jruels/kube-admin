# Setting up HTTP Load Balancing with Ingress
This lab shows how to run a web application behind a Google HTTP load balancer by configuring the Ingress resource.

## Background
GKE offers two types of cloud load balancing to expose your application publicly:

As discussed in pervious labs you can create TCP/UDP load balancers by specifying `type: LoadBalancer` in a Service manifest file. 

The other option is an Ingress controller which offers features like customizable URL maps and TLS termination. Another feature is GKE automatically configures health checks for HTTP(S) load balancers.

## Deploy HTTP web application 
Now let's deploy our demo webserver application. 
```
kubectl apply -f manifests/web-deployment.yaml
```

## Create service
An Ingress still requires a Service which will load balance traffic to all of the pods in the Deployment with matching Selector labels. 
```
kubectl apply -f manifests/web-service.yaml
```

When you create a Service of type `NodePort` with this command, GKE makes your Service available on a randomly- selected port over 30000 on all nodes in the cluster, as discussed.

Verify the Service was created and a node port was allocated:
```
kubectl get svc web
```

Since GKE nodes are not externally accessible by default, creating this Service does not make your application accessible from the Internet.

We need to create an Ingress resource to make web server application publicly available.

## Create an Ingress resource

Ingress is a Kubernetes resource that encapsulates a collection of rules and configuration for routing external HTTP traffic to internal services.

On GKE, Ingress is implemented using Cloud Load Balancing. When you create an Ingress in your cluster, GKE creates an HTTP load balancer and configures it to route traffic to your application.

Create the Ingress 
```
kubectl apply -f manifests/basic-ingress.yaml
```
Once you deploy this manifest, Kubernetes creates an Ingress resource on your cluster. The Ingress controller running in your cluster is responsible for creating an HTTP Load Balancer to route all external HTTP traffic (on port 80) to the web NodePort Service you exposed.

## Visit application

Find out the external IP address of the load balancer serving your application by running:
```
kubectl get ingress basic-ingress
```

Output should look similar to: 
```
NAME            HOSTS     ADDRESS         PORTS     AGE
basic-ingress   *         203.0.113.12    80        2m
```

**Note:**  It may take a few minutes for GKE to allocate an external IP address and set up forwarding rules until the load balancer is ready to serve your application. 

Open a browser to the IP address from command above and you should see a page showing some basic information. 

## Serve multiple applications on Ingress   
You can run multiple services on a single load balancer and public IP by configuring routing rules on the Ingress. By hosting multiple services on the same Ingress, you can avoid creating additional load balancers (which are billable resources) for every Service you expose to the Internet.

Create another web server Deployment with version 2.0 of the same web application.

```
kubectl apply -f manifests/web-deployment-v2.yaml
```

Then, expose the web2 Deployment internally to the cluster on a NodePort Service called web2.   
```
kubectl apply -f manifests/web-service-v2.yaml
```

The following manifest describes an Ingress resource that:   

* routes the requests with path starting with /v2/ to the web2 Service   
* routes all other requests to the web Service   

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: fanout-ingress
spec:
  rules:
  - http:
      paths:
      - path: /*
        backend:
          serviceName: web
          servicePort: 8080
      - path: /v2/*
        backend:
          serviceName: web2
          servicePort: 8080
```

Deploy fanout-ingress
```
kubectl create -f manifests/fanout-ingress.yaml
```

Once the ingress is deployed, find out the public IP address by running:
```
kubectl get ingress fanout-ingress 
```

Then visit the IP address to see that both applications are reachable on the same load balancer:   

* Visit http://[IP_ADDRESS]/ and note that the response contains Version: 1.0.0 (as the request is routed to the web Service)   
* Visit http://[IP_ADDRESS]/v2/ and note that the response contains Version: 2.0.0 (as the request is routed to the web2 Service)   
The only supported wildcard pattern matching for the path field on GKE Ingress is through the * character. For example, you can have rules with path fields like /* or /foo/bar/*.   



