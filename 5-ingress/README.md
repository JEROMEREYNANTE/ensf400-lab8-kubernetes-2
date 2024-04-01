# 5 - Ingresses

## What is an Ingress?

- In Kubernetes, an Ingress is an object that allows access to your Kubernetes services from outside the Kubernetes cluster. You configure access by creating a collection of rules that define which inbound connections reach which services.

- This lets you consolidate your routing rules into a single resource. For example, you might want to send requests to example.com/api/v1/ to an api-v1 service, and requests to example.com/api/v2/ to the api-v2 service. With an Ingress, you can easily set this up without creating a bunch of LoadBalancers or exposing each service on the Node.


### Kubernetes Ingress vs LoadBalancer vs NodePort

These options all do the same thing. They let you expose a service to external network requests. 
They let you send a request from outside the Kubernetes cluster to a service inside the cluster.

### NodePort 

- NodePort is a configuration setting you declare in a service’s YAML. Set the service spec’s type to NodePort. Then, Kubernetes will allocate a specific port on each Node to that service, and any request to your cluster on that port gets forwarded to the service.

- This is cool and easy, it’s just not super robust. You don’t know what port your service is going to be allocated, and the port might get re-allocated at some point.

### LoadBalancer

- You can set a service to be of type LoadBalancer the same way you’d set NodePort— specify the type property in the service’s YAML. There needs to be some external load balancer functionality in the cluster, typically implemented by a cloud provider.

- This is typically heavily dependent on the cloud provider—GKE creates a Network Load Balancer with an IP address that you can use to access your service.

- Every time you want to expose a service to the outside world, you have to create a new LoadBalancer and get an IP address.

## Ingress

- NodePort and LoadBalancer let you expose a service by specifying that value in the service’s type. Ingress, on the other hand, is a completely independent resource to your service. You declare, create and destroy it separately to your services.

- This makes it decoupled and isolated from the services you want to expose. It also helps you to consolidate routing rules into one place.

- The one downside is that you need to configure an Ingress Controller for your cluster. But that’s pretty easy—in this example, we’ll use the Nginx Ingress Controller.

## How to Use Nginx Ingress Controller

- First, enable the ingress add-on for Minikube
 
```bash
$ minikube addons enable ingress
```
 
- Then check that it’s all set up correctly.
 
```bash
$ kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx
NAMESPACE       NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx   ingress-nginx-admission-create-kkcz4        0/1     Completed   0          2m29s
ingress-nginx   ingress-nginx-admission-patch-5g8zb         0/1     Completed   0          2m29s
ingress-nginx   ingress-nginx-controller-7c6974c4d8-d2cmv   1/1     Running     0          2m29s
```
### Creating a Kubernetes Ingress

- First, let’s create two services to demonstrate how the Ingress routes our request. We’ll run two web applications that output a slightly different response.

```bash
$ cd /workspaces/ensf400-lab7-kubernetes-1/5-ingress
$ kubectl apply -f apple.yaml
pod/apple-app created
service/apple-service created

$ kubectl apply -f banana.yaml
pod/banana-app created
service/banana-service created
```

- Create the Ingress in the cluster

```bash
$ kubectl create -f ingress.yaml
```

Perfect! Let’s check that it’s working. If you’re using Minikube, **you might need to replace <Minikube IP> with minikube IP**.
Command to get your minikube IP:
```bash
$ minikube ip
192.168.49.2
```

```
$ curl -kL http://<Minikube IP>/apple
apple

$ curl -kL http://<Minikube IP>/banana
banana

$ curl -kL http://<Minikube IP>/notfound
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

## Ingress Controllers and Ingress Resources

- Kubernetes supports a high level abstraction called Ingress, which allows simple host or URL based HTTP routing. An ingress is a core concept (in beta) of Kubernetes, but is always implemented by a third party proxy. These implementations are known as ingress controllers. An ingress controller is responsible for reading the Ingress Resource information and processing that data accordingly. Different ingress controllers have extended the specification in different ways to support additional use cases.

- Ingress is tightly integrated into Kubernetes, meaning that your existing workflows around kubectl will likely extend nicely to managing ingress. Note that an ingress controller typically doesn’t eliminate the need for an external load balancer — the ingress controller simply adds an additional layer of routing and control behind the load balancer.
