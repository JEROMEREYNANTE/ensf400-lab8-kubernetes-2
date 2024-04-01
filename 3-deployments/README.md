# 3 - Deployments


We looked at ReplicaSets earlier. However, ReplicaSet have one major drawback: 
once you select the pods that are managed by a ReplicaSet, you cannot change their pod templates. 

For example, if you are using a ReplicaSet to deploy four pods with NodeJS running and you want to change the NodeJS image to a newer version, you need to delete the ReplicaSet and recreate it. Restarting the pods causes downtime till the images are available and the pods are running again.

A Deployment resource uses a ReplicaSet to manage the pods. However, it handles updating them in a controlled way. 
Let’s dig deeper into Deployment Controllers and patterns.



## Creating Your First Deployment

The following Deployment definition deploys four pods with nginx as their hosted application:

```bash
$ cd /workspaces/ensf400-lab7-kubernetes-1/3-deployments
$ kubectl create -f nginx-dep.yaml
deployment.apps/nginx-deployment created
```

## Checking the list of application deployment

To list your deployments use the get deployments command:
```bash
$ kubectl get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2/2     2            2           63s

```

```bash
$ kubectl describe deploy
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Sun, 10 Mar 2024 22:31:08 +0000
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=nginx
Replicas:               2 desired | 2 updated | 2 total | 2 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx:1.7.9
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deployment-9d6cbcc65 (2/2 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  5s    deployment-controller  Scaled up replica set nginx-deployment-9d6cbcc65 to 2
  ```

We should have 2 Pods. If not, run the command again. This shows:

    The DESIRED state is showing the configured number of replicas
    The CURRENT state show how many replicas are running now
    The UP-TO-DATE is the number of replicas that were updated to match the desired (configured) state
    The AVAILABLE state shows how many replicas are actually AVAILABLE to the users
    
```bash
$ kubectl get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2/2     2            2           41s
```

```bash
$ kubectl get po
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-9d6cbcc65-bpnvl   1/1     Running   0          52s
nginx-deployment-9d6cbcc65-ff9ml   1/1     Running   0          52s
```

## Step #2. Scale up/down application deployment

Now let’s scale the Deployment to 4 replicas. We are going to use the kubectl scale command,
followed by the deployment type, name and desired number of instances:
```bash
$ kubectl scale deployments/nginx-deployment --replicas=4
deployment.extensions/nginx-deployment scaled
```

The change was applied, and we have 4 instances of the application available. Next, 
let’s check if the number of Pods changed:


Now There should be 4 pods running in the cluster
```bash
$ kubectl get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   4/4     4            4           83s

```

There are 4 Pods now, with different IP addresses. The change was registered in the Deployment events log. To check that, use the describe command:

```bash
$ kubectl describe deployments/nginx-deployment
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Sun, 10 Mar 2024 22:31:08 +0000
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=nginx
Replicas:               4 desired | 4 updated | 4 total | 4 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx:1.7.9
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable
  Available      True    MinimumReplicasAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deployment-9d6cbcc65 (4/4 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  97s   deployment-controller  Scaled up replica set nginx-deployment-9d6cbcc65 to 2
  Normal  ScalingReplicaSet  25s   deployment-controller  Scaled up replica set nginx-deployment-9d6cbcc65 to 4 from 2

```



```
$ kubectl get pods -o wide
NAME                               READY   STATUS    RESTARTS   AGE     IP            NODE       NOMINATED NODE   READINESS GATES
nginx-deployment-9d6cbcc65-bpnvl   1/1     Running   0          2m28s   10.244.0.20   minikube   <none>           <none>
nginx-deployment-9d6cbcc65-ff9ml   1/1     Running   0          2m28s   10.244.0.19   minikube   <none>           <none>
nginx-deployment-9d6cbcc65-nkrrr   1/1     Running   0          76s     10.244.0.22   minikube   <none>           <none>
nginx-deployment-9d6cbcc65-sh5rt   1/1     Running   0          76s     10.244.0.21   minikube   <none>           <none>
```

You can also view in the output of this command that there are 4 replicas now.


# Scaling the service to 2 Replicas 

To scale down the Service to 2 replicas, run again the scale command:

```
$ kubectl scale deployments/nginx-deployment --replicas=2
deployment.extensions/nginx-deployment scaled
$ kubectl get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2/2     2            2           3m2s
```



## Step #3. Perform rolling updates to application deployment

So far, everything our Deployment did is no different than a typical ReplicaSet. The real power of a Deployment lies in its ability to update the pod templates without causing application outage.

Let’s say that you have finished testing the nginx 1.7.9 , and you are ready to use it in production. The current pods are using the older nginx version . The following command changes the deployment pod template to use the new image:

To update the image of the application to new version, use the set image command,
followed by the deployment name and the new image version:

```bash
$ kubectl set image  deployments/nginx-deployment nginx=nginx:1.9.1
deployment.extensions/nginx-deployment image updated
```

The command notified the Deployment to use a different image for your app and initiated a rolling update. Check the status of the new Pods, and view the old one terminating with the get pods command with `-w` option to watch the pod status changes:

```bash
$ kubectl get pods -w
```
(Use `CTRL+C` to terminate watching the pods)

# Checking description of pod again 

```bash
$ kubectl describe pods
Name:             nginx-deployment-7ffd5c8dc9-9c8kz
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.49.2
Start Time:       Sun, 10 Mar 2024 22:37:46 +0000
Labels:           app=nginx
                  pod-template-hash=7ffd5c8dc9
Annotations:      <none>
Status:           Running
IP:               10.244.0.24
IPs:
  IP:           10.244.0.24
Controlled By:  ReplicaSet/nginx-deployment-7ffd5c8dc9
Containers:
  nginx:
    Container ID:   docker://d31fa4ad3d26249d4403dab56cadb23ea227b5e1ab6e623b884bd48b1db07d54
    Image:          nginx:1.9.1
    Image ID:       docker-pullable://nginx@sha256:2f68b99bc0d6d25d0c56876b924ec20418544ff28e1fb89a4c27679a40da811b
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sun, 10 Mar 2024 22:37:47 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-49hrl (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-49hrl:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  16s   default-scheduler  Successfully assigned default/nginx-deployment-7ffd5c8dc9-9c8kz to minikube
  Normal  Pulled     16s   kubelet            Container image "nginx:1.9.1" already present on machine
  Normal  Created    16s   kubelet            Created container nginx
  Normal  Started    16s   kubelet            Started container nginx


Name:             nginx-deployment-7ffd5c8dc9-bzllp
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.49.2
Start Time:       Sun, 10 Mar 2024 22:37:35 +0000
Labels:           app=nginx
                  pod-template-hash=7ffd5c8dc9
Annotations:      <none>
Status:           Running
IP:               10.244.0.23
IPs:
  IP:           10.244.0.23
Controlled By:  ReplicaSet/nginx-deployment-7ffd5c8dc9
Containers:
  nginx:
    Container ID:   docker://f5692da2ba7a54af2d0fee5a3623911507d89df785765d1186bbb90f30f15174
    Image:          nginx:1.9.1
    Image ID:       docker-pullable://nginx@sha256:2f68b99bc0d6d25d0c56876b924ec20418544ff28e1fb89a4c27679a40da811b
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sun, 10 Mar 2024 22:37:45 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-zjw6s (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-zjw6s:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  28s   default-scheduler  Successfully assigned default/nginx-deployment-7ffd5c8dc9-bzllp to minikube
  Normal  Pulling    27s   kubelet            Pulling image "nginx:1.9.1"
  Normal  Pulled     18s   kubelet            Successfully pulled image "nginx:1.9.1" in 9.179s (9.179s including waiting)
  Normal  Created    18s   kubelet            Created container nginx
  Normal  Started    18s   kubelet            Started container nginx
```

## Step #4. Rollback updates to application deployment


The rollout command reverted the deployment to the previous known state. Updates are versioned and you can revert to any previously know state of a Deployment. List again the Pods:

```bash
$ kubectl rollout undo deployments/nginx-deployment
deployment.extensions/nginx-deployment rolled back

$ kubectl rollout status deployments/nginx-deployment 
deployment "nginx-deployment" successfully rolled out

```

After the rollout succeeds, you may want to get the Deployment.

The output shows the update progress until all the pods use the new container image.

The algorithm that Kubernetes Deployments use when deciding how to roll updates is to keep at least 25% of the pods running. Accordingly, it doesn’t kill old pods unless a sufficient number of new ones are up. In the same sense, it does not create new pods until enough pods are no longer running. Through this algorithm, the application is always available during updates.

You can use the following command to determine the update strategy that the Deployment is using:
```
$ kubectl describe deployments | grep Strategy
StrategyType:           RollingUpdate
RollingUpdateStrategy:  25% max unavailable, 25% max surge
```


<!-- ## Step #5. Cleanup

Finally you can clean up the resources you created in your cluster:
```
kubectl delete deployment nginx-deployment
``` -->
