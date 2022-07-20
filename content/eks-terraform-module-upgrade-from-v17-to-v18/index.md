+++
title = "Terraform EKS module upgrade from v17.x to v18.x"
date = 2022-07-20

[taxonomies]
tags = ["AWS", "Terraform", "EKS"]
+++
If you’re like me and you are using the awesome [terraform-aws-eks](https://github.com/terraform-aws-modules/terraform-aws-eks) module to manage your EKS clusters, then you should know that there are many [breaking changes](https://github.com/terraform-aws-modules/terraform-aws-eks/releases/tag/v18.0.0) when upgrading the module version from `v17.x to` `v18.x`, in this guide I will share the steps that I took to ease the migration a little bit (we have more than 12 clusters with similar config) and I hope that this helps someone.

> I highly recommend that you also read the official guide [UPGRADE-18.0.md](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/master/docs/UPGRADE-18.0.md) document
>

## 1) Change module version and variables

Check the variables changes [here](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/master/docs/UPGRADE-18.0.md#variable-and-output-changes), as everyone is using a different configuration, in my case I have ended up with the following:

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"

  # check releases for the latest version
  # https://github.com/terraform-aws-modules/terraform-aws-eks/releases
  version = "18.26.2"

  # Needed for EKS module Upgrade
  # https://github.com/terraform-aws-modules/terraform-aws-eks/blob/master/docs/UPGRADE-18.0.md#upgrade-from-v17x-to-v18x
  prefix_separator                   = ""
  iam_role_name                      = var.cluster_name
  cluster_security_group_name        = var.cluster_name
  cluster_security_group_description = "EKS cluster security group."

  # Add this to avoid issues with AWS Load balancer controller
  # "error":"expect exactly one securityGroup tagged kubernetes.io/cluster/xxx
  # https://github.com/terraform-aws-modules/terraform-aws-eks/issues/1986
  node_security_group_tags = {
    "kubernetes.io/cluster/${var.cluster_name}" = null
  }

  cluster_name                    = local.name
  cluster_version                 = local.cluster_version
  cluster_endpoint_private_access = true
  cluster_endpoint_public_access  = true

  cluster_addons = {
    coredns = {
      resolve_conflicts = "OVERWRITE"
    }
    kube-proxy = {
      resolve_conflicts = "OVERWRITE"
    }
  }

  cluster_encryption_config = [{
    provider_key_arn = aws_kms_key.eks.arn
    resources        = ["secrets"]
  }]

  vpc_id     = module.vpc.vpc_id

  # Rename subnets to subnet_ids
  subnet_ids = module.vpc.private_subnets

  # Rename node_group_defaults to eks_managed_node_group_defaults
  eks_managed_node_group_defaults = {
    ami_type       = "AL2_x86_64"
    instance_types = ["m5.large", "m5a.large", "t3.large", "m5.xlarge"]

    iam_role_attach_cni_policy = true

    update_config = {
      max_unavailable_percentage = 50
    }

    block_device_mappings = {
      xvda = {
        device_name = "/dev/xvda"
        ebs = {
          volume_size = "100"
          volume_type = "gp3"
          encrypted   = true
          kms_key_id  = aws_kms_key.ebs.arn
        }
      }
    }
    metadata_options = {
      http_endpoint               = "enabled"
      http_tokens                 = "required"
      http_put_response_hop_limit = 2
      instance_metadata_tags      = "disabled"
    }
  }
  # Rename eks_managed_node_groups to eks_managed_node_groups
  # the variables in the sub module also changed, so be sure to rename them!
  eks_managed_node_groups = {
    default = {
      name            = "default"
      use_name_prefix = true
      subnet_ids = module.vpc.private_subnets
      desired_size    = 3
      max_size        = 10
      min_size        = 3
      labels = {
        environment = local.environment
        capacity    = "on_demand"
      }
      tags = local.tags
    },
    spot = {
      name            = "spot"
      use_name_prefix = true
      capacity_type   = "SPOT"
      instance_types  = ["m5a.xlarge", "m5.xlarge", "r5.xlarge", "r5a.xlarge"]
      subnet_ids      = module.vpc.private_subnets
      desired_size    = 3
      max_size        = 15
      min_size        = 3
      cluster_version = local.cluster_version

      labels = {
        environment = local.environment
        capacity    = "spot"
      }
      tags = local.tags
  }
  tags = local.tags
}
```

If your `iam_role_name` variable is NOT the cluster_name, the get the cluster iam role name using aws cli:

```bash
export CLUSTER_NAME=bill-eks-test
aws-vault exec aws-prod -- aws eks describe-cluster --name $CLUSTER_NAME --output json | jq -r .cluster.roleArn | cut -d/ -f2
```

## 2) Do a `terraform plan` to see what changes

You should see a lot of resources change including for example IAM, node groups, and even cluster control plane replacement (we will fix that next step)

Since there are a lot of changes and we wanted to make the upgrade simple for us, the best approach is to remove existing node groups and related resources from terraform state, this suggestion also came from users in the [related Github issue](https://github.com/terraform-aws-modules/terraform-aws-eks/issues/1744)

## 3) Terraform state migration and deletion

> Everyone setup is different so be careful when playing with Terrafrom state file, as your tf state might get corrupted, always backup the state file first!
>

```bash
# Backup terraform state first 
terraform state pull > tf_backup.tfstate
# Remove node_groups from tf state
terraform state rm 'module.node_groups'
# Rename the cluster iam role tf resource to new name
terraform state mv 'aws_iam_role.cluster[0]' 'aws_iam_role.this[0]'
# Remove policy attachment for node groups from tf state
terraform state rm 'aws_iam_role_policy_attachment.workers_AmazonEC2ContainerRegistryReadOnly[0]' 'aws_iam_role_policy_attachment.workers_AmazonEKSWorkerNodePolicy[0]' 'aws_iam_role_policy_attachment.workers_AmazonEKS_CNI_Policy[0]'
# Remove node groups security group from state
terraform state rm 'aws_security_group.workers[0]'
# Remove node groups aws_security_group_rule resources from tf state
terraform state rm $(terraform state list | grep aws_security_group_rule.workers)
# Remove cluster aws_security_group_rule resources from tf state
terraform state rm $(terraform state list | grep aws_security_group_rule.cluster)
# Remove IAM Role of node groups from tf state
terraform state rm 'aws_iam_role.workers[0]'

# If you are managing the aws-auth configmap using EKS module
# Then remove aws-auth configmap from tf state as now the module dropped the support
terraform state rm 'kubernetes_config_map.aws_auth[0]'

# If you use addons then remove from state as well
terraform state rm 'aws_eks_addon.kube_proxy' 'aws_eks_addon.vpc_cni' 'aws_eks_addon.coredns'
# Then import the new add ons to state
terraform import 'aws_eks_addon.this["coredns"]' cluster:coredns
terraform import 'aws_eks_addon.this["vpc-cni"]' cluster:vpc-cni
terraform import 'aws_eks_addon.this["kube-proxy"]' cluster:kube-proxy
```

## 4) Test `terraform plan` again

### Make sure you DO NOT see the following

1. The cluster Must be replaced
2. Old node groups must be destroyed
3. Node groups related resources (sg, sg rules, old node iam) must be destroyed

### You should see the following:

1. Some cluster policy changes.
2. New node group adding.
3. New node groups related resources.

## 5) Proceed with `terraform apply`

You can now start by applying the changes with `terraform apply` .

After the `apply` is complete, you should see new nodes joining the cluster and working as expected

## 6) Manually delete the old node group and related resources

Now you can go ahead and delete the old node group manually and pods will restart on new node groups 👍 🎉.

Other related resources should be manually deleted as well such as old node group security groups.

# Over to you

Like this post? Consider following me on Medium **[billhegazy](https://billhegazy.medium.com/).**

If you have questions or want to reach out, add me on **[LinkedIn](https://www.linkedin.com/in/bhegazy/)**.

*Originally published at [https://billhegazy.com](https://billhegazy.com/aws-solution-architect-professional-certificate/).*