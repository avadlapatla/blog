# Kubelet Customization on Managed Node Groups using eksctl

Amazon EKS managed node groups automates the provisioning and lifecycle management of Kubernetes nodes for you. Recently support for custom AMI and launch template has been added to the feature set [Launch Announcement](https://aws.amazon.com/blogs/containers/introducing-launch-template-and-custom-ami-support-in-amazon-eks-managed-node-groups/) .

eksctl has been updated to support this feature for managed node group. Kubelet args can be passed to bootstrap script via --kubelet-extra-args using **overrideBootstrapCommand** field. However, this command can only be set when a custom AMI is specified, so for this example we will just use EKS Optimized AMI.

```
  - name: managed-ng-2
    ami: ami-05bc8cd159ecc4bcb
    minSize: 1
    desiredCapacity: 1
    maxSize: 4
    instanceType: t3.medium
    preBootstrapCommands:
    - yum install -y amazon-ssm-agent
    - systemctl enable amazon-ssm-agent && systemctl start amazon-ssm-agent
    labels:
      role: worker
    tags:
      nodegroup-name: managed-ng-2
    privateNetworking: true
    overrideBootstrapCommand: |
      /etc/eks/bootstrap.sh west-example --kubelet-extra-args '--serialize-image-pulls=true --cpu-cfs-quota=true'
    iam:
      attachPolicyARNs:
      - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
      - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
      - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
      - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
```

Complete eksctl yaml configuration file can be downloaded from here [cluster.yaml](https://github.com/avadlapatla/blog/blob/master/docs/files/cluster.yaml){target=_blank}