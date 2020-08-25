#Deploying Cluster Autoscaler on EKS Fargate

```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: west-example
  region: us-west-2

iam:
  withOIDC: true
  serviceAccounts:
  - metadata:
      name: cluster-autoscaler
      namespace: kube-system
    attachPolicy: # inline policy can be defined along with `attachPolicyARNs`
      Version: "2012-10-17"
      Statement:
      - Effect: Allow
        Action:
        - "autoscaling:DescribeAutoScalingGroups"
        - "autoscaling:DescribeAutoScalingInstances"
        - "autoscaling:DescribeLaunchConfigurations"
        - "autoscaling:DescribeTags"
        - "ec2:DescribeLaunchTemplateVersions"
        Resource: '*'
      - Effect: Allow
        Action:
        - "autoscaling:SetDesiredCapacity"
        - "autoscaling:TerminateInstanceInAutoScalingGroup"
        Resource: '*'
        Condition:
          StringEquals:
            autoscaling:ResourceTag/k8s.io/cluster-autoscaler/enabled : true
            autoscaling:ResourceTag/k8s.io/cluster-autoscaler/west-example:
              - owned
              - shared

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

fargateProfiles:
  - name: fp-kube-system
    selectors:
      # All workloads in the "dev" Kubernetes namespace matching the following
      # label selectors will be scheduled onto Fargate:
      - namespace: kube-system
        labels:
          runtime: fargate

```

Save the above configuration as cluster.yaml

### Fargate Profile

The above eksctl configuration creates fargate profile named fp-kube-system. Pods that are run in the kube-system namespace and have the labels defined in the configuration will be scheduled on fargate. To create the fargate profile run the following command.

```
eksctl create fargateprofile -f cluster.yaml
```

### Associate IAM OIDC Provider

Run the following command to create IAM OIDC identity provider for EKS cluster

```
eksctl utils associate-iam-oidc-provider -f cluster.yaml
```

### IAM Roles for Service Account

eksctl makes it easy to create IAM roles for service account by define the policies in the configuration file. Run the following command to create cluster-autoscaler IAM role for service account, the policies attached to the service account will be leveraged by cluster autoscaler to auto discover and manage autoscaling groups.

```
eksctl create iamserviceaccount -f cluster.yaml --override-existing-serviceaccounts
```

### Cluster Autoscaler Configurtion

Download the cluster autoscaler example deployment file from [here](https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml)

Replace the deployment object from the above file with the following content

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
        runtime: fargate
      annotations:
        prometheus.io/scrape: 'true'
        prometheus.io/port: '8085'
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
        - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.17.3
          name: cluster-autoscaler
          resources:
            limits:
              cpu: 100m
              memory: 300Mi
            requests:
              cpu: 100m
              memory: 300Mi
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/west-example
          volumeMounts:
            - name: ssl-certs
              mountPath: /etc/ssl/certs/ca-certificates.crt
              subPath: ca-certificates
              readOnly: true
          imagePullPolicy: "Always"
          env:
            - name: AWS_REGION
              value: us-west-2
      volumes:
        - name: ssl-certs
          configMap:
            name: ssl-certs
```

We are changing the way the cluster autoscaler is deployed on the EKS cluster. The deployment spec above leverages ssl-certs config map to mount ca-certificates. It also uses the latest container image matching the kubernetes version and cluster-autoscaler service account.

### Create ssl-certs configmap

Cluster autoscaler requires ca-certificates, we will deploy ca-certificates as configmap and mount the configmap as volume in our deployment.

The ca-certificates file can be downloded from here [ssl-certs](../files/ca-certificates.crt) 

*Note ca-certificates file has been extracted from the pod running on Amazon Linux 2 (/etc/ssl/certs/ca-bundle.crt)

Run the following command to create ssl-certs configmap

```
kubectl create secret -n kube-system ssl-certs --from-file=ca-certificates=ca-certificates.crt
```

### Deploy Cluster Autoscaler

Deploy cluster autoscaler using the following command

```
kubectl apply -f cluster-autoscaler-autodiscover.yaml
```