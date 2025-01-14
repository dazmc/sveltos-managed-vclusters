# Sveltos vCluster Automation on EKS

This repo contains everything you'll need to test out automting the installation and management of vCluster instances on EKS, using Sveltos. All you need is an AWS account and a Kind cluster.

You can also opt to install the vClusters in a different host cluster, but just note that the K8s configs in this repo have only been tested on EKS. There may be subtle changes that you'll have to handle on your own.

## Standing up the K8s clusters

You'll need two clusters. I used Kind to create a management cluster, and EKS for my host cluster.

### Management Cluster in Kind

This step is pretty simple as long as you have [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/) installed:

```sh
kind create cluster --name management
```

This will add the `kind-management` cluster and context to your default Kubeconfig file, as well as set it as the current context.

From there, you can install Sveltos by following their installation instructions: https://projectsveltos.github.io/sveltos/getting_started/install/install/.

### EKS Cluster

The files in `tofu-eks/` come from [this repo](https://github.com/hashicorp-education/learn-terraform-provision-eks-cluster), which has the MPL 2.0 license. For the most part the files are the same, however I'm using OpenTofu instead of Terraform.

Using the AWS CLI, configure your local profile to target an AWS account with the appropriate permissions needed to set up EKS, VPC, IAM, etc. If you're unsure, admin-level permissions work as long as you understand the risks of provisioning that much power to your user account.

With the local profile set, you can run:

```sh
cd tofu-eks
tofu init
tofu plan --out plan
tofu apply plan
```

That should stand up your EKS cluster. Just know that it can take a few minutes for everything to come online.

You can add the EKS cluster to your default Kubeconfig by running:

```sh
aws eks update-kubeconfig --region <region-code> --name <cluster-name>
```

This will use the active AWS profile to generate a token for authn/authz every time you run `kubectl` against the EKS context.

It will also set the EKS cluster as your current context.

### EKS Default Storage Class

**This is super important!!! If you don't do this, your vCluster pods will be stuck in a Pending state!!!**

One thing I noticed about these .tf files is that they fail to set the EBS storage class as the default. I'll admit, there's probably a configuration you can set very quickly to make that happen, but I haven't looked that up yet.

What I did instead was I added the right annotation to the StorageClass YAML that sets EBS as the default.

You can add the following to the `metadata.annotations` block of the EBS Storage Class configuration:

```yaml
annotations:
  # ...
  "storageclass.kubernetes.io/is-default-class": true
```

### Register that EKS cluster in Sveltos

You'll need to create a ServiceAccount and corresponding token Secret that Sveltos can use to access the EKS cluster's control plane API.

If you're not familiar with how to do that, this page on Oracle's website is a good reference: https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengaddingserviceaccttoken.htm.

Then it's a matter of filling in the blanks in a standard Kubeconfig template: https://devopscube.com/kubernetes-kubeconfig-file/

P.S. I'm sure there's an automated way to do this, but the 30 seconds of copy-pasta has never been painful enough for me to bother learning how.

## EVENTS!!!

Now the fun part. Change your `kubectl` context back to `kind-management`, and run the following from the root of this repo:

```sh
# applies the event source for load balancer creation
kubectl apply -f sveltos-resources/lb
# applies the event source for exported secret creation
kubectl apply -f sveltos-resources/vcluster
```

You're crushing it.

From here, all you have to do is run the single `helm` command that kicks off the entire automated workflow.

```sh
helm upgrade --install developer-load-balancers ./charts
```

Bangarang, Rufio.
