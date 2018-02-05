# cloud-platforms-k8s-kops
Resources for Kubernetes clusters with Kops

## Prerequisites
- Install [Kops](https://github.com/kubernetes/kops)
- Install [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/)
- Install [git-crypt](https://www.agwa.name/projects/git-crypt/)

**GPG Note:** As this repo uses `git-crypt` your GPG key must be added to this repo for you to be able to use the SSH keys, but is not required to obtain Kubernetes credentials for exist as long as you have valid AWS credentials, as they can be obtained from S3.

### MacOS
```
brew install kops
brew install kubernetes-cli
brew install git-crypt
```

## Working with existing cluster (`cluster1`)

### Prerequisites
1. Make sure you have credentials for the correct AWS account and that they are active:
 `$ export AWS_PROFILE=mojds-platforms-integration`
2. Kops stores resource specifications and credentials in an S3 bucket, which must be specified before using `kops`:
 `$ export KOPS_STATE_STORE=s3://moj-cloud-platforms-kops-state-store`
3. Download cluster credentials from S3 and configure `kubectl` for use:
 `kops export kubecfg cluster-1.kops.integration.dsd.io`
4. Verify that you can interact with the cluster:
 `$ kubectl cluster-info`
5. Check what's running:
 `$ kubectl get pods --all-namespaces`

## Editing cluster config / changing config

Refer to [kops documentation](https://github.com/kubernetes/kops/blob/master/docs/changing_configuration.md)

## Creating a new cluster

### Create cluster specification
Generate a new SSH key and use `kops` to generate a new cluster specification - example for `cluster1.yaml`:

```
export KOPS_STATE_STORE=s3://moj-cloud-platforms-kops-state-store
export CLUSTER_NAME=cluster1

ssh-keygen -t rsa -f ssh/${CLUSTER_NAME}_kops_id_rsa -N '' -C ${CLUSTER_NAME}

kops create cluster \
    --name ${CLUSTER_NAME}.kops.integration.dsd.io \
    --zones=eu-west-1a,eu-west-1b,eu-west-1c \
    --node-size=t2.medium \
    --node-count=3 \
    --master-zones=eu-west-1a,eu-west-1b,eu-west-1c \
    --master-size=t2.medium \
    --topology=private \
    --dns-zone=kops.integration.dsd.io \
    --ssh-public-key=ssh/${CLUSTER_NAME}_kops_id_rsa.pub \
    --authorization=RBAC \
    --networking=calico \
    --output=yaml \
    --bastion \
    --dry-run \
    > ${CLUSTER_NAME}.yaml
```

### Create SSH public key in kops state store
`$ kops create secret --name ${CLUSTER_NAME}.kops.integration.dsd.io sshpublickey admin -i ssh/${CLUSTER_NAME}_kops_id_rsa.pub`

### Create cluster specification in kops state store
`$ kops create -f ${CLUSTER_NAME}.yaml`

### Create cluster resources in AWS
aka update cluster in AWS according to the yaml specification:
`$ kops update cluster ${CLUSTER_NAME}.kops.integration.dsd.io --yes`

It takes a few minutes for the cluster to deploy and come up - you can check progress with:

`$ kops validate cluster`

Once it reports `Your cluster ${CLUSTER_NAME}.kops.integration.dsd.io is ready` you can proceed to use `kubectl` to interact with the cluster.