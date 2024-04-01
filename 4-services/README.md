# What are Kubernetes Services?

Say, you have pods running nginx in a flat, cluster wide, address space. In theory, you could talk to these pods directly, but what happens when a node dies? The pods die with it, and the Deployment will create new ones, with different IPs. This is the problem a Service solves.


Kubernetes Pods are mortal. They are born and when they die, they are not resurrected. If you use a Deployment to run your app, it can create and destroy Pods dynamically. Each Pod gets its own IP address, however in a Deployment, the set of Pods running in one moment in time could be different from the set of Pods running that application a moment later.

This leads to a problem: if some set of Pods (call them “backends”) provides functionality to other Pods (call them “frontends”) inside your cluster, how do the frontends find out and keep track of which IP address to connect to, so that the frontend can use the backend part of the workload?

Enter Services


A Kubernetes Service is an abstraction which defines a logical set of Pods running somewhere in your cluster, that all provide the same functionality. When created, each Service is assigned a unique IP address (also called clusterIP). This address is tied to the lifespan of the Service, and will not change while the Service is alive. Pods can be configured to talk to the Service, and know that communication to the Service will be automatically load-balanced out to some pod that is a member of the Service.

## Deploying a Kubernetes Service

Like all other Kubernetes objects, a Service can be defined using a YAML or JSON file that contains the necessary definitions (they can also be created using just the command line, but this is not the recommended practice). Let’s create a NodeJS service definition. It may look like the following:

```bash
$ cd /workspaces/ensf400-lab7-kubernetes-1/4-services
$ kubectl apply -f nginx-svc.yaml
service/my-nginx created
```

This specification will create a Service which targets TCP port 80 on any Pod with the `run: my-nginx` label, and expose it on an abstracted Service port (targetPort: is the port the container accepts traffic on, port: is the abstracted Service port, which can be any port other pods use to access the Service). View Service API object to see the list of supported fields in service definition. Check your Service

```bash
$ kubectl get svc my-nginx
NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
my-nginx   ClusterIP   10.98.222.214   <none>        80/TCP    46s
```

As mentioned previously, a Service is backed by a group of Pods. These Pods are exposed through endpoints. The Service’s selector will be evaluated continuously and the results will be POSTed to an Endpoints object also named my-nginx. When a Pod dies, it is automatically removed from the endpoints, and new Pods matching the Service’s selector will automatically get added to the endpoints. Check the endpoints, and note that the IPs are the same as the Pods created in the first step:


```bash
$ kubectl describe svc my-nginx
Name:              my-nginx
Namespace:         default
Labels:            run=my-nginx
Annotations:       <none>
Selector:          run=my-nginx
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.98.222.214
IPs:               10.98.222.214
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.244.0.27:80,10.244.0.28:80
Session Affinity:  None
Events:            <none>
```

You should now be able to curl the nginx Service on <CLUSTER-IP>:<PORT> from any node in your cluster. Note that the Service IP is completely virtual, it never hits the wire. If you’re curious about how this works you can read more about the service proxy.

## Accessing the Service

Kubernetes supports 2 primary modes of finding a Service - environment variables and DNS

### Environment Variables

When a Pod runs on a Node, the kubelet adds a set of environment variables for each active Service. This introduces an ordering problem. To see why, inspect the environment of your running nginx Pods (your Pod name will be different):

```bash
$ kubectl exec nginx-deployment-6bc585fddc-hlsht -- printenv | grep SERVICE
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT_HTTPS=443
```
(NOTE: the pod name may be different because of the random string in the end. Use `kubectl get po` and pick your own valid pod name.)


Note there’s no mention of your Service. This is because you created the replicas before the Service. Another disadvantage of doing this is that the scheduler might put both Pods on the same machine, which will take your entire Service down if it dies. We can do this the right way by killing the 2 Pods and waiting for the Deployment to recreate them. This time around the Service exists before the replicas. This will give you scheduler-level Service spreading of your Pods (provided all your nodes have equal capacity), as well as the right environment variables:

```bash
$ kubectl scale deployment nginx-deployment --replicas=0
deployment.apps/nginx-deployment scaled
$ kubectl scale deployment nginx-deployment --replicas=2
deployment.apps/nginx-deployment scaled
```

```bash
kubectl get pods -l run=my-nginx -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP            NODE       NOMINATED NODE   READINESS GATES
nginx-deployment-6bc585fddc-cfs8b   1/1     Running   0          7s    10.244.0.30   minikube   <none>           <none>
nginx-deployment-6bc585fddc-pdlph   1/1     Running   0          9s    10.244.0.29   minikube   <none>           <none>
```

You may notice that the pods have different names, since they are killed and recreated.

```bash
kubectl exec nginx-deployment-6bc585fddc-cfs8b -- printenv | grep SERVICE
```
(NOTE: the pod name may be different because of the random string in the end. Use `kubectl get po` and pick your own valid pod name.)


```bash
MY_NGINX_SERVICE_PORT=80
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
MY_NGINX_SERVICE_HOST=10.98.222.214
KUBERNETES_SERVICE_HOST=10.96.0.1
```


### DNS

Kubernetes offers a DNS cluster addon Service that automatically assigns dns names to other Services. You can check if it’s running on your cluster:

```bash
$ kubectl get services kube-dns --namespace=kube-system
```

```bash
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   3h53m
```

The rest of this section will assume you have a Service with a long lived IP (my-nginx), and a DNS server that has assigned a name to that IP. Here we use the CoreDNS cluster addon (application name kube-dns), so you can talk to the Service from any pod in your cluster using standard methods (e.g. gethostbyname()). If CoreDNS isn’t running, you can enable it referring to the CoreDNS README or Installing CoreDNS. Let’s run another curl application to test this:

```bash
kubectl run curl --image=radial/busyboxplus:curl -i --tty
```

```bash
If you don't see a command prompt, try pressing enter.

Then, hit enter and run nslookup my-nginx:

[ root@curl-131556218-9fnch:/ ]$ nslookup my-nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      my-nginx
Address 1: 10.98.222.214 my-nginx.default.svc.cluster.local
```
Type `exit` to exist the terminal of the curl pod.


## Exposing the Service

For some parts of your applications you may want to expose a Service onto an external IP address. Kubernetes supports two ways of doing this: NodePorts and LoadBalancers. The Service created in the last section already used NodePort, so your nginx HTTPS replica is ready to serve traffic on the internet if your node has a public IP.

```bash
$ kubectl apply -f nginx-svc-nodeport.yaml 
service/my-nginx configured
$ kubectl get svc my-nginx -o yaml | grep nodePort -C 5
```

```yaml
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: http
    nodePort: 31704
    port: 8080
    protocol: TCP
    targetPort: 80
  - name: https
    nodePort: 32453
    port: 443
    protocol: TCP
    targetPort: 443
  selector:
    run: my-nginx
```

Check the service again and you will see the port mappings for the service:
```bash
$ kubectl get svc my-nginx
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                        AGE
my-nginx     NodePort    10.98.222.214   <none>        8080:31704/TCP,443:32453/TCP   22m
```

```bash
$ kubectl get nodes -o yaml | grep addresses -C 1
  status:
    addresses:
    - address: 192.168.49.2
...
```
Use the IP above (`192.168.49.2` in this example, your might be different), and the two node ports configured (`31704`) to test the HTTP connectivity.
```bash
$ curl http://192.168.49.2:31704 -k
...
<h1>Welcome to nginx!</h1>
...
```

