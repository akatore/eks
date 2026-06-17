Using **`eksctl`** (which builds the cluster via CloudFormation) or deploy a **CloudFormation template** directly.

The fastest way to spin this up using `eksctl` while adhering to their rules:

### Step 1: Gather Your Prerequisites

Before creating the cluster, you need your AWS Account ID and the subnets from your default VPC.

**Get your Account ID:**

```bash
aws sts get-caller-identity --query "Account" --output text

```
<img width="366" height="28" alt="image" src="https://github.com/user-attachments/assets/85e6cc11-c2d6-4c3d-9900-d83bb87baf52" />


**Get your Subnet IDs:**

```bash
aws ec2 describe-subnets --query "Subnets[*].SubnetId" --output text

```
<img width="900" height="39" alt="image" src="https://github.com/user-attachments/assets/bd648cbb-f583-4136-ad19-5b47a710c2b9" />


*(Note down at least two subnet IDs for the next steps).*

### Step 2: Create a cluster config file (`cluster.yaml`)

`eksctl` uses a configuration file to define everything at once. Create a file named `cluster.yaml` and paste the following into it. (Make sure to swap in your `<ACCOUNT_ID>` and `<SUBNET_IDs>`):

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eks-cluster-stack # Using the exact name required by the playground
  region: us-east-1

vpc:
  subnets:
    public:
      us-east-1a: { id: <SUBNET_1> }
      us-east-1b: { id: <SUBNET_2> }

iam:
  serviceRoleARN: "arn:aws:iam::<ACCOUNT_ID>:role/eksClusterRole"

managedNodeGroups:
  - name: playground-nodes
    instanceType: t3.micro
    minSize: 1
    maxSize: 3 # Hard limit set by KodeKloud
    desiredCapacity: 2
    iam:
      instanceRoleARN: "arn:aws:iam::<ACCOUNT_ID>:role/AmazonEKSNodeRole"

```

### Step 3: Launch with eksctl

Run the following command to provision the cluster. `eksctl` will package this into a CloudFormation stack, which should bypass the direct API restriction:

```bash
eksctl create cluster -f cluster.yaml

```

*Note: This process usually takes around 15–20 minutes as CloudFormation provisions the control plane and nodes.*

Let me know if `eksctl` throws any stack naming errors, as we can easily pivot to a raw CloudFormation YAML template if the playground is extremely strict about the exact stack string!
