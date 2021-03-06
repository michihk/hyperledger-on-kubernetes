# Creating an EKS cluster

You have a number of options when creating the Kubernetes cluster in which you'll deploy Hyperledger Fabric:
 
* The easiest option is to use eksctl to create an AWS EKS (Elastic Container Service for Kubernetes) cluster
* You can use KOPS to create a cluster: https://github.com/kubernetes/kops
* Or you can use an existing Kubernetes cluster, as long as you can SSH into the worker nodes

Whichever method you choose, you will need SSH access to the Kubernetes worker nodes in order to install EFS utils. 
EFS utils is used to mount the EFS (Elastic File System) used to store the Hyperledger Fabric CA certs/keys. 
Instructions on how to do this are in the section 'Install EFS utils on each Kubernetes worker node' below.

This README will focus on creating an EKS cluster using eksctl.

## Creating an EKS cluster

There are two methods you can use to create an EKS cluster. You can:
 
* create the cluster from your own laptop
* create a Cloud9 environment, and create the cluster from Cloud9

I suggest you use Cloud9 as it comes pre-installed with packages that we'll need. However, if you have a Mac and prefer
to use it, the instructions below should still work for you. The instructions have not been fully tested on Windows.

## Potential Issue

If you see the message below when running `eksctl`, you'll need to pass the list of availability zones to eksctl, as 
shown below. I have only seen this issue when creating an EKS cluster in us-east-1.

```bash
2018-08-14T02:44:28Z [✖]  unable to create cluster control plane: UnsupportedAvailabilityZoneException: Cannot create cluster 'eks-fabric' because us-east-1e, the targeted availability zone, does not currently have sufficient capacity to support the cluster. Retry and choose from these availability zones: us-east-1a, us-east-1b, us-east-1d
        status code: 400, request id: f69224d4-9f6b-11e8-a5af-3d2857a10e45
```

Fix this by specifying the AZs you want to use. You'll need to delete the cluster first:

```bash
eksctl delete cluster --name eks-fabric
```

Then create the cluster:

```bash
eksctl create cluster --ssh-public-key <YOUR AWS KEYPAIR> --name eks-fabric --region us-east-1 --kubeconfig=./kubeconfig.eks-fabric.yaml --zones=us-east-1a,us-east-1b,us-east-1d
```

## Creating an EKS cluster using Cloud9

Read all the instructions in this section before you create your Cloud9 instance. There are a couple of items we need
to address as you follow the tutorial below.

To create an EKS cluster using Cloud9, follow the instructions 
[here](https://github.com/pahud/amazon-eks-workshop/blob/master/00-getting-started/create-eks-with-eksctl.md). 

Step 7: You'll execute `aws configure` to configure credentials. The credentials you enter here are the AWS access key 
and secret key that belong to the AWS account you will use to create your EKS cluster, and to run your Cloud9 IDE. There 
are two ways to obtain the AWS access key and secret key:

* if you use the AWS CLI, on your laptop you should have an `.aws` directory. On Mac, it's under your home directory, 
i.e. `~/.aws`. This contains a credentials and config file. The credentials file contains your AWS keys. You can copy the 
AWS access key and secret key from here
* otherwise, log in to the AWS IAM console, find your user and click the `Security Credentials` tab. You can create a new access key here
and copy the AWS access key and secret key into the `aws configure` command

Step 11:

* Create a keypair, using the details in the section below titled 'Creating a keypair'
* Use this statement to create the EKS cluster instead of the one in Step 11:

```bash
eksctl create cluster --ssh-public-key <YOUR AWS KEYPAIR> --name eks-fabric --region us-west-2 --kubeconfig=./kubeconfig.eks-fabric.yaml
```

Note that `ssh-public-key` is the name of your keypair (i.e. the value you passed to `aws ec2 create-key-pair --key-name <KEYNAME>`), 
not the path to the .pem file you saved.

Now go an get a cup of coffee. It will take around 10-25 minutes to create the EKS cluster. 

Once the cluster creation is complete, you should see something like this.

```bash
$ eksctl create cluster --ssh-public-key eks-c9-keypair --name eks-fabric --region us-west-2 --kubeconfig=./kubeconfig.eks-fabric.yaml
2018-08-12T03:47:45Z [ℹ]  setting availability zones to [us-west-2b us-west-2a us-west-2c]
2018-08-12T03:47:45Z [ℹ]  SSH public key file "eks-c9-keypair" does not exist; will assume existing EC2 key pair
2018-08-12T03:47:45Z [ℹ]  found EC2 key pair "eks-c9-keypair"
2018-08-12T03:47:45Z [ℹ]  creating EKS cluster "eks-fabric" in "us-west-2" region
2018-08-12T03:47:45Z [ℹ]  creating ServiceRole stack "EKS-eks-fabric-ServiceRole"
2018-08-12T03:47:45Z [ℹ]  creating VPC stack "EKS-eks-fabric-VPC"
2018-08-12T03:48:26Z [✔]  created ServiceRole stack "EKS-eks-fabric-ServiceRole"
2018-08-12T03:49:06Z [✔]  created VPC stack "EKS-eks-fabric-VPC"
2018-08-12T03:49:06Z [ℹ]  creating control plane "eks-fabric"
2018-08-12T03:58:27Z [✔]  created control plane "eks-fabric"
2018-08-12T03:58:27Z [ℹ]  creating DefaultNodeGroup stack "EKS-eks-fabric-DefaultNodeGroup"
2018-08-12T04:02:08Z [✔]  created DefaultNodeGroup stack "EKS-eks-fabric-DefaultNodeGroup"
2018-08-12T04:02:08Z [✔]  all EKS cluster "eks-fabric" resources has been created
2018-08-12T04:02:08Z [✔]  saved kubeconfig as "./kubeconfig.eks-fabric.yaml"
2018-08-12T04:02:13Z [ℹ]  the cluster has 0 nodes
2018-08-12T04:02:13Z [ℹ]  waiting for at least 2 nodes to become ready
2018-08-12T04:02:44Z [ℹ]  the cluster has 2 nodes
2018-08-12T04:02:44Z [ℹ]  node "ip-192-168-171-30.us-west-2.compute.internal" is ready
2018-08-12T04:02:44Z [ℹ]  node "ip-192-168-214-46.us-west-2.compute.internal" is ready
2018-08-12T04:02:45Z [ℹ]  kubectl command should work with "./kubeconfig.eks-fabric.yaml", try 'kubectl --kubeconfig=./kubeconfig.eks-fabric.yaml get nodes'
2018-08-12T04:02:45Z [✔]  EKS cluster "eks-fabric" in "us-west-2" region is ready
```

Now check whether `kubectl` can access your Kubernetes cluster:

```bash
$ kubectl --kubeconfig=./kubeconfig.eks-fabric.yaml get nodes
NAME                                           STATUS    ROLES     AGE       VERSION
ip-192-168-171-30.us-west-2.compute.internal   Ready     <none>    5m        v1.10.3
ip-192-168-214-46.us-west-2.compute.internal   Ready     <none>    5m        v1.10.3
```

Once the cluster is created, follow the instructions in the section 'Install EFS utils on each Kubernetes worker node' below.

## Creating an EKS cluster using your Mac

The instructions below are similar to those on the eksctl website, with additional enhancements to the 'create cluster' 
command.

To download the latest release, run:

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

Alternatively, macOS users can use Homebrew:

```bash
brew install weaveworks/tap/eksctl
```

You will need to have AWS API credentials configured. What works for AWS CLI or any other tools (kops, Terraform etc), should be 
sufficient. You can use [~/.aws/credentials file](https://docs.aws.amazon.com/cli/latest/userguide/cli-config-files.html) 
or [environment variables](https://docs.aws.amazon.com/cli/latest/userguide/cli-environment.html). For more information 
read the [AWS documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-environment.html).

If you used 'brew' to install eksctl, it should automatically install the Heptio authenticator, which is required
to access EKS using kubectl. If you used the 'curl' method above, install the Heptio authenticator manually:

```bash
curl -o heptio-authenticator-aws https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-06-05/bin/linux/amd64/heptio-authenticator-aws && chmod +x heptio-authenticator-aws && mkdir ~/bin && export PATH=~/bin:$PATH && echo 'export PATH=~/bin:$PATH' >> ~/.bashrc && mv heptio-authenticator-aws ~/bin/
```

Create a keypair, then create the EKS cluster:

* Create a keypair, using the details in the section below titled 'Creating a keypair'
* Use this statement to create the EKS cluster instead of the one in Step 11:

```bash
eksctl create cluster --ssh-public-key <YOUR AWS KEYPAIR> --name eks-fabric --region us-west-2 --kubeconfig=./kubeconfig.eks-fabric.yaml
```

Note that `ssh-public-key` is the name of your public key (i.e. the value you passed to `aws ec2 create-key-pair --key-name <KEYNAME>`), 
not the path to the .pem file you saved.

Now go an get a cup of coffee. It will take around 10-25 minutes to create the EKS cluster. 

Once the cluster creation is complete, you should see something like this.

```bash
$ eksctl create cluster --ssh-public-key eksctl --name eks-fabric --region us-west-2 --kubeconfig=./kubeconfig.eks-fabric.yaml --profile account2
2018-08-04T17:23:54+08:00 [ℹ]  setting availability zones to [us-west-2a us-west-2b us-west-2c]
2018-08-04T17:23:54+08:00 [ℹ]  SSH public key file "eksctl" does not exist; will assume existing EC2 key pair
2018-08-04T17:23:55+08:00 [ℹ]  found EC2 key pair "eksctl"
2018-08-04T17:23:55+08:00 [ℹ]  creating EKS cluster "eks-fabric" in "us-west-2" region
2018-08-04T17:23:55+08:00 [ℹ]  creating VPC stack "EKS-eks-fabric-VPC"
2018-08-04T17:23:55+08:00 [ℹ]  creating ServiceRole stack "EKS-eks-fabric-ServiceRole"
2018-08-04T17:24:36+08:00 [✔]  created ServiceRole stack "EKS-eks-fabric-ServiceRole"
^[[A^[[B2018-08-04T17:24:56+08:00 [✔]  created VPC stack "EKS-eks-fabric-VPC"
2018-08-04T17:24:56+08:00 [ℹ]  creating control plane "eks-fabric"
2018-08-04T17:35:39+08:00 [✔]  created control plane "eks-fabric"
2018-08-04T17:35:39+08:00 [ℹ]  creating DefaultNodeGroup stack "EKS-eks-fabric-DefaultNodeGroup"
2018-08-04T17:49:42+08:00 [✔]  created DefaultNodeGroup stack "EKS-eks-fabric-DefaultNodeGroup"
2018-08-04T17:49:42+08:00 [✔]  all EKS cluster "eks-fabric" resources has been created
2018-08-04T17:49:42+08:00 [✔]  saved kubeconfig as "./kubeconfig.eks-fabric.yaml"
2018-08-04T17:49:44+08:00 [ℹ]  the cluster has 1 nodes
2018-08-04T17:49:44+08:00 [ℹ]  node "ip-192-168-113-140.us-west-2.compute.internal" is not ready
2018-08-04T17:49:44+08:00 [ℹ]  waiting for at least 2 nodes to become ready
2018-08-04T17:50:11+08:00 [ℹ]  the cluster has 2 nodes
2018-08-04T17:50:11+08:00 [ℹ]  node "ip-192-168-113-140.us-west-2.compute.internal" is ready
2018-08-04T17:50:11+08:00 [ℹ]  node "ip-192-168-163-2.us-west-2.compute.internal" is ready
2018-08-04T17:50:13+08:00 [ℹ]  kubectl command should work with "./kubeconfig.eks-fabric.yaml", try 'kubectl --kubeconfig=./kubeconfig.eks-fabric.yaml get nodes'
2018-08-04T17:50:13+08:00 [✔]  EKS cluster "eks-fabric" in "us-west-2" region is ready
```

Now check whether `kubectl` can access your Kubernetes cluster:

```bash
$ kubectl --kubeconfig=./kubeconfig.eks-fabric.yaml get nodes
NAME                                            STATUS    ROLES     AGE       VERSION
ip-192-168-113-140.us-west-2.compute.internal   Ready     <none>    2m        v1.10.3
ip-192-168-163-2.us-west-2.compute.internal     Ready     <none>    2m        v1.10.3
```

Once the cluster is created, follow the instructions in the section 'Install EFS utils on each Kubernetes worker node' below.

## Install EFS utils on each Kubernetes worker node 

EFS utils is required on each Kubernetes worker node to enable the worker to mount the EFS used to store the Hyperledger 
Fabric CA certs/keys.

The EC2 instances should be easy to identify in the AWS console as the instance name is prefixed with your EKS cluster name, e.g.
`eks-fabric-default-Node`. Clicking on each instance will show you the security group and the public DNS, which you'll need
when you SSH into the instance.

Before we can SSH into the Kubernetes worker nodes, we need to update the security group to allow SSH ingress.

You can do this from the console following the instructions here: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/authorizing-access-to-an-instance.html#add-rule-authorize-access
If you'll SSH from your Mac, you can update the security group to allow SSH from your IP only. If you'll SSH from Cloud9,
the quickest way is to allow SSH to all. (if someone has a more secure idea, please let me know).

Now SSH into each worker node and install the EFS utils, using the keypair you created earlier, as follows:

```bash
ssh ec2-user@ec2-54-244-69-254.us-west-2.compute.amazonaws.com -i eks-c9-keypair.pem
```

If you see a `Permission denied` message, with details indicating `Permissions 0664 for 'eks-c9-keypair.pem' are too open`,
you'll need to `chmod` your key as follows:

```bash
chmod 400 eks-c9-keypair.pem
```

After SSH'ing into the worker node, run:

```bash
sudo yum install -y amazon-efs-utils
```

Type `exit` to exit the EC2 instance. Install the EFS utils on all your EKS worker nodes. If you following the 
instructions here you should have 2 nodes.

## Creating a keypair

You will need a keypair in the same region as you create the EKS cluster. We default this to us-west-2, so create a
keypair in us-west-2 and make sure you download the keypair to either your laptop or your Cloud9 environment, depending
on which method you will use to create your EKS cluster.

Make sure your AWS CLI is pointing to the right account and region.

See [Using Key Pairs](https://docs.aws.amazon.com/cli/latest/userguide/cli-ec2-keypairs.html) in the AWS documentation
for details on how to use the command line to create and download a keypair.

Make sure you following the instructions to `chmod 400` your key.
