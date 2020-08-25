# Deploying EKS Cluster using eksctl

###Prerequisites
1. Getting started with eksctl [[Instructions](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html)]


###Setup
EKSCTL supports creating and managing clusters leveraging YAML files. Using the following YAML configuration file, eksctl creates the cluster named west-example in the us-west-2 region. It then provisions EKS managed node group with SSM agent pre installed.

```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: west-example
  region: us-west-2


managedNodeGroups:
  - name: managed-ng-1
    minSize: 2
    desiredCapacity: 2
    maxSize: 4
    instanceType: t3.medium
    preBootstrapCommands:
    - yum install -y amazon-ssm-agent
    - systemctl enable amazon-ssm-agent && systemctl start amazon-ssm-agent
    labels:
      role: worker
    tags:
      nodegroup-name: managed-ng-1
    privateNetworking: true
    iam:
      attachPolicyARNs:
      - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
      - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
      - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
      - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
```

Copy the above configuration to a file named cluster.yaml and use the below command to provision the cluster and managed node groups

```
eksctl create cluster -f cluster.yaml
```