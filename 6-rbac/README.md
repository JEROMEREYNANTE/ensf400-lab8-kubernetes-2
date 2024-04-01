# 6 - Role-Based Access Control (RBAC)


RBAC is a security design that restricts access to valuable resources based on the role the user holds, hence the name role-based. To understand the importance and the need of having RBAC policies in place, let’s consider a system that doesn’t use it. Let’s say that you have an HR management solution, but the only security access measure used is that users must authenticate themselves through a username and a password. Having provided their credentials, users gain full access to every module in the system (recruitment, training, staff performance, salaries, etc.). A slightly more secure system will differentiate between regular user access and “admin” access, with the latter providing potentially destructive privileges. For example, ordinary users cannot delete a module from the system, whereas an administrator can. But still, users without admin access can read and modify the module’s data regardless of whether their current job entails doing this.

If you worked as a Linux administrator for any length of time, you appreciate the importance of having a security system that implements a security matrix of access and authority. In the old days of Linux and UNIX, you could either be a “normal” user with minimal access to the system resources, or you can have “root” access. Root access virtually gives you full control over the machine that you can accidentally bring the whole system down. Needless to say that if an intruder could gain access to this root account, your entire system is at high risk. Accordingly, RBAC systems were introduced.

In a system that uses RBAC, there is minimal mention of the “superuser” or the administrator who has access to everything. Instead, there’s more reference to the access level, the role, and the privilege. Even administrators can be categorized based on their job requirements. So, backup administrators should have full access to the tools that they use to do backups. But they shouldn’t be able to stop the webserver or change the system’s date and time.


## Creating Role and Role Binding

A Role in Kubernetes is as a Group in other RBAC implementations. Instead of defining different authorization rules for each user, you attach those rules to a group and add users to it. When users resign, for example, you only need to remove them from one place. Similarly, when a new user joins the company or gets transferred to another department, you need to change the roles they’re associated with.

First, create a service account for testing the role binding:

```bash
$ kubectl apply -f service-account-div.yaml 
serviceaccount/div created
```

Let’s create a role that enables our user to execute the get pods command:

```bash
$ kubectl apply -f role.yaml
role.rbac.authorization.k8s.io/get-pods created
```

Now, we have a role that enables its users to list the pods on the default namespace. But, in order for the div user to be able to execute the get pods, it needs to get bound to this role.

Kubernetes offers the RoleBinding resource to link roles with their objects (for example, users). Let’s add role-binding.yaml file to look as follows:

```bash
$ kubectl apply -f role-binding.yaml
rolebinding.rbac.authorization.k8s.io/div-get-pods created
```

Create a pod using the service account `div` we just created. Note the following two lines spefifying the service account for the pod:

```yaml
spec:
  ...
  serviceAccountName: div
  automountServiceAccountToken: false
  ...
```

```bash
$ kubectl apply -f pod01.yaml
pod/webserver created
```
Check if the pod is using the designated service account `div`.

```bash
$ kubectl describe pod webserver 
Name:             webserver
Namespace:        default
Priority:         0
Service Account:  div
Node:             minikube/192.168.49.2
...
```
