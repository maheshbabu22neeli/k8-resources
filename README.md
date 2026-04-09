# k8-resources
Learning kubernetes resources


![k8-workstation-flow.drawio.svg](images/k8-workstation-flow.drawio.svg)

## Launch EC2 Instance(k8-workstation)
Launch EC2 instance with volume size 50 GB

##  Extend the disk
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





