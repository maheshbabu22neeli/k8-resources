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
kubectl apply -f 03-labels.yaml
kubectl describe pod pod-labels -n roboshop
kubectl delete -f 03-labels.yaml
```

## Annotations
- Annotations are similar like labels add metadata, but annotations are used to select external resources like load-balancers.
- Where labels have limitations in key value length and size, whereas in annotations no such limit
- Annotations are used to select external services
```shell
kubectl apply -f 04-annotations.yaml
kubectl describe pod pod-annotations -n roboshop
kubectl delete -f 04-annotations.yaml
```

## Resources
- In Kubernetes, containers are free to use the required resources like memory
- Memory will increase or decrease based on usage, but this might be a good practice we need to restrict the resources.
- When the utility is more, we have to achieve by using auto-scaling.
````shell
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
````

