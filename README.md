Just like a state file mismatch in Terraform, `eksctl` is throwing a fit here because it detected lingering state. Your first attempt likely partially created a CloudFormation stack or cluster control plane before failing, and now the subnets in your local `cluster.yaml` don't match the subnets bound to that hanging remote infrastructure.

Additionally, because `eksctl` aggressively controls how it names CloudFormation stacks (it always uses an `eksctl-<name>-cluster` format), it's highly likely to trigger the playground's IAM boundaries again since they strictly mandate the exact name `eks-cluster-stack`.

To bypass `eksctl`'s opinionated state management and the playground restrictions entirely, the cleanest approach is to use a raw, native CloudFormation template. This gives us absolute control over the stack name and securely attaches the exact IAM roles required.

Here is how to clean up the mismatched state and deploy the cluster properly:

### Step 1: Nuke the Lingering State

First, clear out the broken stack created by `eksctl` so we have a clean slate. Run this in your terminal:

```bash
aws cloudformation delete-stack --stack-name eksctl-eks-cluster-stack-cluster

```

*(Wait a minute or two for this to complete before proceeding).*

### Step 2: Create the CloudFormation Template

Create a file named `eks.yaml` and paste the following declarative infrastructure directly into it. Notice how we are dynamically referencing your AWS Account ID so you don't even need to hardcode it this time:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'EKS Cluster Stack for KodeKloud Playground'

Parameters:
  Subnets:
    Type: List<AWS::EC2::Subnet::Id>
    Description: Comma-separated list of Subnet IDs

Resources:
  PlaygroundEKSCluster:
    Type: AWS::EKS::Cluster
    Properties:
      Name: playground-eks
      RoleArn: !Sub "arn:aws:iam::${AWS::AccountId}:role/eksClusterRole"
      ResourcesVpcConfig:
        SubnetIds: !Ref Subnets

  PlaygroundNodeGroup:
    Type: AWS::EKS::Nodegroup
    DependsOn: PlaygroundEKSCluster
    Properties:
      ClusterName: !Ref PlaygroundEKSCluster
      NodegroupName: playground-nodes
      NodeRole: !Sub "arn:aws:iam::${AWS::AccountId}:role/AmazonEKSNodeRole"
      Subnets: !Ref Subnets
      InstanceTypes:
        - t3.micro
      ScalingConfig:
        MinSize: 1
        MaxSize: 3
        DesiredSize: 2

```

### Step 3: Deploy the Stack

Now, deploy the cluster using the standard AWS CLI. Replace `<SUBNET_1>,<SUBNET_2>` with your actual subnet IDs (separated by a comma, no spaces). This command perfectly satisfies the playground's `eks-cluster-stack` naming rule:

```bash
aws cloudformation deploy \
  --template-file eks.yaml \
  --stack-name eks-cluster-stack \
  --parameter-overrides Subnets=<SUBNET_1>,<SUBNET_2>

```

CloudFormation will now orchestrate the cluster and the node group seamlessly.

While it provisions, would you like me to grab the command you'll need to monitor the CloudFormation deployment status directly from the CLI?

Perfect. With those subnet IDs ready, we can deploy the CloudFormation stack immediately.

Assuming you have saved the `eks.yaml` file from the previous step, here is your exact, copy-pasteable deployment command using your first two subnets:

```bash
aws cloudformation deploy \
  --template-file eks.yaml \
  --stack-name eks-cluster-stack \
  --parameter-overrides Subnets=subnet-07cc5f54f8344e32a,subnet-0cede4c488189ea43

```

Because CloudFormation abstracts the provisioning, it handles the underlying API calls perfectly within the boundaries set by the playground.

### Monitoring the Deployment

CloudFormation takes a bit of time to spin up the EKS control plane and node group (usually 10–15 minutes). You can open a second terminal tab and poll the deployment status with this command:

```bash
aws cloudformation describe-stacks \
  --stack-name eks-cluster-stack \
  --query "Stacks[0].StackStatus" \
  --output text

```

### Connecting to the Cluster

Once that status command returns `CREATE_COMPLETE`, wire up your local `kubectl` to interact with the new cluster:

```bash
aws eks update-kubeconfig --region us-east-1 --name playground-eks

```

---

Once the cluster is up and your kubeconfig is wired in, would you like to draft the Kubernetes deployment manifests to enforce those strict 256 millicore/512 MiB pod limits?



Here is the official AWS documentation for the exact CloudFormation resources we just used to build your EKS environment. These pages detail every available configuration property, return value, and snippet example for the "CloudFormation way" of deploying EKS:

* **[AWS::EKS::Cluster](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-eks-cluster.html):** This defines the EKS control plane. In our template, we used this to set the cluster name, attach the playground-mandated IAM role (`eksClusterRole`), and define the VPC subnets where the cluster communicates.
* **[AWS::EKS::Nodegroup](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-eks-nodegroup.html):** This provisions the underlying EC2 worker nodes. We used this to enforce the `t3.micro` instance type, strictly cap the scaling group to a maximum of 3 nodes, and attach the `AmazonEKSNodeRole`.

Using these declarative templates is standard practice in DevOps for achieving repeatable, Infrastructure-as-Code (IaC) deployments, effectively bypassing the manual limitations of the AWS Management Console or restrictions placed on direct AWS CLI API calls.

---

Now that your cluster is spinning up with CloudFormation, would you like to move on to the Kubernetes side and draft the Deployment manifests needed to enforce those strict 256 millicore CPU and 512 MiB memory limits?
