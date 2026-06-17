Starting with an **automated approach using the AWS CLI** rather than manual click-ops in the console.

Building your infrastructure via code or scripts gives you a reproducible, production-ready environment and avoids the risk of accidentally clicking the wrong configuration. In this specific KodeKloud playground, using the AWS CLI is also the cleanest way to bypass the strict `eks-cluster-stack` CloudFormation naming rule, while securely attaching the exact IAM roles they require.

Here is how you can set this up using the AWS CLI, ensuring you stay well within the playground's boundaries.

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

### Step 2: Create the EKS Cluster

Use the CLI to create the cluster, pointing explicitly to the allowed `eksClusterRole`. Replace `<ACCOUNT_ID>` and `<SUBNET_1>,<SUBNET_2>` with your values.

```bash
aws eks create-cluster \
  --name my-eks-cluster \
  --role-arn arn:aws:iam::<ACCOUNT_ID>:role/eksClusterRole \
  --resources-vpc-config subnetIds=<SUBNET_1>,<SUBNET_2>

```
<img width="479" height="699" alt="image" src="https://github.com/user-attachments/assets/d46efab3-da9f-43d0-933a-f11d1222867c" />


*Note: Cluster creation usually takes 10–15 minutes. You can check its status using `aws eks describe-cluster --name my-eks-cluster --query "cluster.status"` until it says `ACTIVE`.*
<img width="421" height="24" alt="image" src="https://github.com/user-attachments/assets/2666187a-aebb-4d5d-9ba9-d6c1fd7d8dc3" />

### Step 3: Create the Node Group

Once the cluster is active, attach your EC2 nodes. This command enforces the maximum of 3 nodes, uses an allowed instance type (`t3.micro`), and attaches the mandatory `AmazonEKSNodeRole`.

```bash
aws eks create-nodegroup \
  --cluster-name my-eks-cluster \
  --nodegroup-name playground-nodes \
  --node-role arn:aws:iam::<ACCOUNT_ID>:role/AmazonEKSNodeRole \
  --subnets <SUBNET_1> <SUBNET_2> \
  --instance-types t3.micro \
  --scaling-config minSize=1,maxSize=3,desiredSize=2

```

### Step 4: Update Your Kubeconfig

Once the node group is ready, connect your local `kubectl` to the new cluster:

```bash
aws eks update-kubeconfig --region us-east-1 --name my-eks-cluster

```

---

Do you already have your VPC Subnet IDs ready in your playground terminal, or would you like me to walk through setting up a custom VPC configuration first?
