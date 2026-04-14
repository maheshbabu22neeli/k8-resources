# k8-resources
Learning kubernetes resources


![k8-workstation-flow.drawio.svg](images/k8-workstation-flow.drawio.svg)

## Launch EC2 Instance(k8-workstation)
Launch EC2 instance with volume size 50 GB

##  Extend disk
- To utilise maximum performance
```shell
sudo growpart /dev/nvme0n1 4
sudo lvextend -L +30G /dev/RootVG/varVol
sudo xfs_growfs /var
df -hT
```

## Install Docker in k8-workstation
- To build images, push images to ECR/Docker hub
```shell
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user
```

## Install Eks-Ctl in k8-workstation
- Install eks-ctl which is used to install EKS cluster
- We can create, delete and update EKS Cluster.

```shell
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo install -m 0755 /tmp/eksctl /usr/local/bin && rm /tmp/eksctl

Now, verify version

[ ec2-user@ip-172-31-18-254 ~ ]$ eksctl version
0.225.0
```

## Install Kube-Ctl in k8-workstation
Install kube-ctl to connect with EKS cluster
```shell
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.35.2/2026-02-27/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH

Now, verify version

[ ec2-user@ip-172-31-18-254 ~ ]$ kubectl version
Client Version: v1.35.2-eks-f69f56f
Kustomize Version: v5.7.1
The connection to the server localhost:8080 was refused - did you specify the right host or port?

[ ec2-user@ip-172-31-18-254 ~ ]$ kubectl version --client
Client Version: v1.35.2-eks-f69f56f
Kustomize Version: v5.7.1
```

## Configure AWS CLI
- To provide authentication while creating eks cluster, connecting to cluster
```shell
aws configure
```

## Install Kubernetes Cluster
- Using eks-ctl we can install EKS Cluster
- You can find the code in `https://github.com/maheshbabu22neeli/k8-eksctl`

## Install K9S 
```shell

curl -sS https://webinstall.dev/k9s | bash

```

## Namespace
- K8 is PaaS
- It is an isolated space inside k8-cluster, where we can create our project related resources. We can completely control them. 
- It is like our project inside k8s.
- whatever the manifest file, we see `kind` and `apiVersion`. 
- `kind` tells what resource we are going to build
- `apiVersion` tells what version we are about to give. 
```shell
[ ec2-user@ip-172-31-19-208 ~/k8-eksctl ]$ kubectl get namespace
NAME              STATUS   AGE
default           Active   13m
kube-node-lease   Active   13m
kube-public       Active   13m
kube-system       Active   13m


We can create our own namespace
[ ec2-user@ip-172-31-19-208 ~/k8-eksctl ]$ kubectl create namespace roboshop

We can delete using 
[ ec2-user@ip-172-31-19-208 ~/k8-eksctl ]$ kubectl delete namespace roboshop

We can create using yaml file as well (we genuinely use this yamls files for k8 resource creations)
kubectl apply -f 01-namespace.yaml
kubectl describe ns roboshop

kubectl config set-context --current --namespace=roboshop

kubectl delete -f 01-namespace.yaml
```

## POD
- POD is smallest deployable unit in kubernetes
- A POD can have multiple containers in it.
- Each container in POD share same OS and network
- Multi containers are used mainly to push the logs from the same storage/network to ELK
- The extra container is called side-car container
```shell
kubectl apply -f 02-pod.yaml
kubectl get pods -n roboshop
kubectl describe pod nginx -n roboshop
kubectl get pods -o wide -n roboshop
kubectl get pod nginx -o wide -n roboshop
kubectl delete -f 02-pod.yaml
kubectl exec -it nginx -n roboshop -- bash

kubectl exec -it multi-container -c nginx -n roboshop -- bash   (on multi containers)
kubectl exec -it multi-container -c almalinux -n roboshop -- bash   (on multi containers)
```

## Labels
- Labels add metadata to the resource
- Labels are used as selectors in kubernetes
- Special characters are not allowed in labels
- Labels are to select internal services
```shell
kubectl apply -f 04-labels.yaml
kubectl describe pod pod-labels -n roboshop
kubectl delete -f 04-labels.yaml
```

## Annotations
- Annotations are similar like labels add metadata, but annotations are used to select external resources like load-balancers.
- Where labels have limitations in key value length and size, whereas in annotations no such limit
- Annotations are used to select external services
```shell
kubectl apply -f 05-annotations.yaml
kubectl describe pod pod-annotations -n roboshop
kubectl delete -f 05-annotations.yaml
```

## Resources
- In Kubernetes, containers are free to use the required resources like memory
- Memory will increase or decrease based on usage, but this might be a good practice we need to restrict the resources.
- When the utility is more, we have to achieve by using auto-scaling.
````shell
kubectl apply -f 06-resources.yaml
kubectl describe pod resources -n roboshop
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      # this is minimum resource, soft limit
      requests:
        memory: "64Mi"  # RAM
        cpu: "100m"
        
      # this is maximum resource, hard limit
      limits:
        memory: "128Mi"  # RAM
        cpu: "150m"
        
$ kubectl describe pod resources -n roboshop
Name:             resources
------
------
------
------
Containers:
  nginx:
    Container ID:   containerd://a9bea7dd58c2b163708
    Image:          nginx
    Restart Count:  0
    Limits:
      cpu:     150m
      memory:  128Mi
    Requests:
      cpu:        100m
      memory:     64Mi

kubectl delete -f 06-resources.yaml
````

## ENV
- Environment variables are useful for pods
```shell
spec:
  containers:
  - name: nginx
    image: nginx
    env:
    - name: project
      value: roboshop
    - name: component
      value: frontend
    - name: environment
      value: dev
      
kubectl apply -f 07-env.yaml
kubectl get pods -n roboshop
kubectl exec -it pod-env -n roboshop -- bash
root@pod-env:/# env
project=roboshop
HOSTNAME=pod-env
PWD=/
HOME=/root
environment=dev
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
component=frontend
_=/usr/bin/env

kubectl delete -f 07-env.yaml
```

## Config-Map
- For simple two to three key value pairs we can use ENV, but what if we have 100 key value pairs.
- This can be resolved by Config-Map and attach that to POD.
````shell
kubectl apply -f 08-configmap.yaml
$ kubectl get cm
NAME               DATA   AGE
kube-root-ca.crt   1      131m
nginx-configmap    3      8s

$ kubectl describe cm nginx-configmap
Name:         nginx-configmap
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
course:
----
kubernetes

environment:
----
dev

project:
----
roboshop

````
```shell
How to Test the confimap by attaching to POD, refer 09-pod-configmap.yaml

spec:
  containers:
  - name: nginx
    image: nginx
    envFrom:
    - configMapRef:
        name: nginx-configmap
        
kubectl apply -f 09-pod-configmap.yaml
kubectl get pods -n roboshop
kubectl exec -it pod-configmap-test -n roboshop -- bash
````


## Service
- If we want POD to POD communication, IP address of POD's are not useful as they are ephemeral (temporary).
- We have to use k8 services to achieve pod to pod communication.
- Load-balancing can also be achieved through services
- If pod1 wants to communicate with pod2 it goes via service. ex: pod1 -> service -> pod2
- Service Types
  - Cluster IP (by default)
  - NodePort
  - Load balancer

### Cluster IP
![ClusterIP_Service.drawio.svg](images/ClusterIP_Service.drawio.svg)
- Cluster IP is for the pod's to communicate internally with in the cluster
```shell
kubectl apply -f 01-namespace.yaml
kubectl apply -f 04-labels.yaml
kubectl apply -f 10-service.yaml
$ kubectl describe svc nginx-service -n roboshop
Name:                     nginx-service
Namespace:                roboshop
Labels:                   <none>
Annotations:              <none>
Selector:                 component=frontend,environment=dev,project=roboshop
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.100.234.214
IPs:                      10.100.234.214
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
Endpoints:                192.168.58.161:80
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>

kubectl apply -f 11-service-test.yaml

~/k8-resources ]$ kubectl get pods -n roboshop
NAME           READY   STATUS    RESTARTS   AGE
pod-labels     1/1     Running   0          30s
service-test   1/1     Running   0          13s

$ kubectl exec -it service-test -n roboshop -- bash
[root@service-test /]# curl nginx-service
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, nginx is successfully installed and working.
Further configuration is required for the web server, reverse proxy,
API gateway, load balancer, content cache, or other features.</p>

<p>For online documentation and support please refer to
<a href="https://nginx.org/">nginx.org</a>.<br/>
To engage with the community please visit
<a href="https://community.nginx.org/">community.nginx.org</a>.<br/>
For enterprise grade support, professional services, additional
security features and capabilities please refer to
<a href="https://f5.com/nginx">f5.com/nginx</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
[root@service-test /]#
```

### Node Port

![NodePort_Service.drawio.svg](images/NodePort_Service.drawio.svg)

- Node Port is to communicate external with the cluster
- Which means an external user want to communicate with our application.
- ClusterIP is a subset of NodePort

````shell
kubectl apply -f 04-labels.yaml
kubectl apply -f 12-service-nodeport.yaml

kubectl get svc -n roboshop
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-service   NodePort   10.100.16.90   <none>        80:30303/TCP   47s
````
- Now open 0.0.0.0/0 for 30303 port to one of the NodeGroup instances (EC2)
- Take the public IP of the NodeGroup instance and open in browser `http://54.227.110.173:30303/`
- Browser -> NodePort Service -> Labels POD

### Load Balancer
![Loadbalancer_Service.drawio.svg](images/Loadbalancer_Service.drawio.svg)
- Exposing an IP address to access our application using NodePort service is not safe.
- Exposing an IP address is a security threat.
- This can avoid by using Load balancer to access our application.
- This creates the load-balancer and adds EC2 NodeGroups as a target instances.
- This adds the load-balancer SG to EC2 NodeGroups SG's to allow the traffic
- Take DNS(Details Tab) from the load balancer and access through browser `http://a994af23a3a8-1766.us-east-1.elb.amazonaws.com/.
- NodePort is a subset of LoadBalancer
```shell
kubectl apply -f 13-service-loadbalancer.yaml

$ kubectl get svc -n roboshop
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                                     PORT(S)        AGE
nginx-service   LoadBalancer   10.100.85.187   a994af23a3a8-1766.us-east-1.elb.amazonaws.com   80:31678/TCP   12s

kubectl delete -f 13-service-loadbalancer.yaml
```

## Sets
In Sets, we have 
1. Replica Set
2. Deployment Set
3. Daemon Set
4. StatefulSet

### ReplicaSet
- Replica Set will create multiple replicas of the pod.
- It always runs the desired number of pod's
- Even if we delete any one pod from the desired number of pod's using `kubectl delete pod <pod-name> - n roboshop`, it automatically creates the other pod to match the replica set.
- Every time replicaset make sure that desired number of pods are running.
```shell
kubectl apply -f 14-replicaset.yaml

kubectl get rs -n roboshop

$ kubectl get pods -n roboshop
NAME          READY   STATUS    RESTARTS   AGE
nginx-8m8x8   1/1     Running   0          10s
nginx-fb7cm   1/1     Running   0          10s
nginx-z6s7l   1/1     Running   0          10s

```

### DeploymentSet
- In replica set if we want to upgrade the image version, it unable to update and not able to create a new pod. 
- The only job of replica-set is to maintain the desired number of pods.

- In Deployment, We can own replica-set and pods creation
- In Deployment, you don't have to worry about managing the ReplicaSets
- It is recommended to use Deployments when you want ReplicaSets
- Deployments as a mechanism to orchestrate Pod creation, deletion and updates
- Using deployment, we can update the pod to new version without deleting the existing resources(old replica-set / old pods)
- We can define Deployments to create new ReplicaSets, or to remove existing Deployments and adopt all their resources with new Deployments

- Container is a subset of Pod
- Pod is subset of replica-set
- Replica-Set is a subset of deployment
- 

```shell
kubectl apply -f 15-deploymentset.yaml

$ kubectl get deployment -n roboshop
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   3/3     3            3           60s

$ kubectl get rs -n roboshop
NAME               DESIRED   CURRENT   READY   AGE
nginx-5bd7bc7c5b   3         3         3       22s

$ kubectl get pods -n roboshop
NAME                     READY   STATUS    RESTARTS   AGE
nginx-5bd7bc7c5b-66d7w   1/1     Running   0          99s
nginx-5bd7bc7c5b-dmlcl   1/1     Running   0          99s
nginx-5bd7bc7c5b-sp98s   1/1     Running   0          99s

After image version updated.

    spec:
      containers:
        - name: nginx
          #image: nginx
          # Updating to a new image or new version
          image: nginx:alpine3.23
          
Replica-Set Updated
$ kubectl get rs -n roboshop
NAME               DESIRED   CURRENT   READY   AGE
nginx-5bd7bc7c5b   0         0         0       6m15s
nginx-74b96d5c65   3         3         3       22s

Pods updated, which means new version deployed automatically by removing the old version pods.
$ kubectl get pods -n roboshop
NAME                     READY   STATUS    RESTARTS   AGE
nginx-74b96d5c65-4gg7j   1/1     Running   0          70s
nginx-74b96d5c65-mmc54   1/1     Running   0          65s
nginx-74b96d5c65-wtb46   1/1     Running   0          67s


Assume, in our new deployment we found that a bug and application is not working as expected.
In this case, we need to rollback the application to old version. 
And this can be easy in kubernetes using below command

$ kubectl rollout history deployment/nginx -n roboshop
deployment.apps/nginx
REVISION  CHANGE-CAUSE
1         <none>
2         Updated image to nginx:alpine3.23

We found two revisions.

$ kubectl rollout undo deployment/nginx -n roboshop
deployment.apps/nginx rolled back

$ kubectl get pods -n roboshop
NAME                     READY   STATUS    RESTARTS   AGE
nginx-5bd7bc7c5b-89lj8   1/1     Running   0          11s
nginx-5bd7bc7c5b-mvqt2   1/1     Running   0          14s
nginx-5bd7bc7c5b-vfkxv   1/1     Running   0          17s

We have previous version pods and rollout successfully
```