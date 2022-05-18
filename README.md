Kubernetes on AWS with kOpS
--------------------------------------------

Kops is great. You could set-up a Kubernetes cluster in AWS with a single command line.  Kops manage declaratively the state of a Kubernetes cluster using Cluster API, which inspired Kubernetes [Cluster API](https://cluster-api.sigs.k8s.io/). 

Like any other solution , kops is highly opinionated and not for everyone. It's main strength is a granular control of every aspect of deployed Kubernetes components - API, kubelet, ETCD, CNI, etc  

The process outlined in this project has been tested successfully with k8s v1.17.13. It is easy to adjust the tempates to newer versions of k8s.

Check kops documentation for more details: https://kops.sigs.k8s.io/

# Requirements

## kops version
Version 1.18.0

## AWS account access
Ensure you have a local AWS profile configured (aws configure --profile myprofile) with sufficeint privileges to 
manage all required resources - EC2, EBS, ELB, DNS zone, etc.

## Provisioned AWS VPC 
Provision a VPC with 3 private + 3 public subnets in 3 AZs using a Terraform module like this one
https://github.com/terraform-aws-modules/terraform-aws-vpc

Kubernetes is deployed in the private subnets for security reasons. This implies that a VPN access is required to the k8s API server. You might want to try AWS VPN as a simple to deploy solution to access resources in the private subnets.

Make sure you enter the correct VPC and subnet IDs in the value file - [values_files/values-us-east-1.yaml](values_files/values-us-east-1.yaml).

## An AWS S3 bucket to manage state.

It is recommended the KOPS_STATE_STORE path is unique per k8s cluster.
```
export KOPS_STATE_STORE=s3://s3-state-bucket/kops
export NAME=mars-us-east-1.dev.example.com
```

## aws-iam-authenticator
With this version, one needs to manually configure aws-auth ConfigMap in kube-system namespace to make failing aws-iam-authenticator working. In newer kops version this is not longer necessary. 

## Dex with Gangway 
The templates configure the API server to integrate with Dex. Remove or adjust the OIDC settings as needed.
Using Dex is recommended. It simplifies the onboarding of large number of users with the cluster.
```shell
kubeAPIServer:
    # https://github.com/heptiolabs/gangway
    # https://github.com/dexidp/dex/blob/master/Documentation/kubernetes.md
    oidcIssuerURL: https://dex.example.com/dex
    oidcClientID: gangway
    oidcUsernameClaim: email
    oidcGroupsClaim: groups
```    

# K8s provisioning

The provisioning ie management of the k8s cluster consists of two stepts
- Create the clsuter spec file from templates, snippets and values files.
- Create/update cluster state using the cluster spec file.

Dir layout
```shell
 tree .
.
├── docs
│   ├── authentication-iam.md
│   └── dex-sso.md
├── README.md
├── snippets
│   ├── audit.yaml
│   ├── bashrc
│   ├── coredns.cfg
│   ├── masters.json
│   ├── nodes.json
│   └── packages.yaml
├── templates
│   ├── tm-mars-ci-nodes.yaml
│   └── tm-mars-mixed-instances.yaml
├── tm-cluster.yaml -> templates/tm-mars-ci-nodes.yaml
└── values_files
    └── values-us-east-1.yaml

```

## kops templates

The `tm-cluster.yml` sym link is used to point to the currently used template.

Make sure tm-cluster points to the correct 
1. [templates/tm-mars-mixed-instances.yaml](templates/tm-mars-mixed-instances.yaml)
Mixed EC2 instances with the whole cluster deployed in a single AZ for performance and reduced cost. Uncomment other sub-zones to expand the cluster across 3 AZs.

2. [templates/tm-mars-ci-nodes.yaml](templates/tm-mars-ci-nodes.yaml)
Mixed EC2 instances with the addition of dedicated InstanceGroup for a CI service like Concourse CI.

For more details on Mixed Instances https://kops.sigs.k8s.io/instance_groups/#mixedinstancespolicy-aws-only

Add EC2 Spot instances to save on cost.

## Snippets 
Various segments that are too big and unwieldy to manage in the yaml cluster spec. They are imported as snippets which are easier to validate and review.

[CoreDNS snippet](snippets/coredns.cfg) snippet demonstrates how to customize CoreDNS to manage forwarding to a custom private zone `foo.example.org`. Remove it if you don't need it.

In case of using local node DNS caching one would need to customize its settings to reflect CoreDNS setup.

## Provisioning

Create kops cluster spec from a template
```
kops toolbox template \
--template tm-cluster.yaml \
--values values_files/values-us-east-1.yaml \
--snippets snippets \
--output cluster.yaml \
--name $NAME
```

## If state/cluster doesn't exist
```
kops create --name $NAME -f cluster.yaml
```

## If state/cluster already exists
```
kops replace -f cluster.yaml
```

## Upgrade the cluster
In case of updating cluster version, need to updade the state,
run upgrade command first and then follow up with the next steps:
```
kops upgrade cluster --name $NAME --yes
```

## Update the cluster
See the differences with existing state
```
kops update cluster --name $NAME
```

Apply changes
```
kops update cluster --name $NAME --yes
```

## Roll out changes
See which InstanceGroup needs updates and a restart.
```
kops rolling-update cluster --name $NAME
```

Rollout changes
```
kops rolling-update cluster --name $NAME --yes
```

Apply changes without validating the cluster (aws client only).
K8s nodes will not be drained before applying changes.
```
kops rolling-update cluster --name $NAME --cloudonly --yes
```

## Validate cluster
```
kops validate cluster
```


# Importing Kubernetes config from existing cluster
In case someone else provisioned the cluster, import k8s config with kops:
```
kops export kubecfg
```

# Delete a Cluster
To completely delete the cluster and its state:
First run in preview mode to list what will be deleted.
```
kops delete cluster --name $NAME
```

Then do it for real:
```
kops delete cluster --name $NAME --yes
```

# Customization
Consder using GitOps services like FluxCD to install all components like ingress controllers, service mesh controllers, monitoring, cert management, etc to satisfy users requirements.