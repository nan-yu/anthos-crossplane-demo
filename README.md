
#### Referenced videos:
#### - Crossplane - GitOps-based Infrastructure as Code through Kubernetes API: https://youtu.be/n8KjVmuHm7A
#### - How to apply GitOps to everything - combining Argo CD and Crossplane: https://youtu.be/yrj4lmScKHQ
#### - K3d - How to run Kubernetes cluster locally using Rancher k3s: https://youtu.be/mCesuGk-Fks


### Setup 

```sh
git clone https://github.com/vfarcic/crossplane-composite-demo.git

cd crossplane-composite-demo

cp cluster-orig.yaml cluster.yaml
```

# Install Crossplane CLI from https://crossplane.io/docs/v1.3/getting-started/install-configure.html#start-with-a-self-hosted-crossplane

### Please watch https://youtu.be/mCesuGk-Fks if you are not familiar with k3d
### From any K8s platform
```sh

kubectl create namespace team-a

helm repo add crossplane-stable \
    https://charts.crossplane.io/stable

helm repo update

helm upgrade --install \
    crossplane crossplane-stable/crossplane \
    --namespace crossplane-system \
    --create-namespace \
    --wait

kubectl apply --filename definition.yaml
```
#### Run the rest of the setup instructions specific to your provider, or, if you are brave, for all of them.

## Setup GCP #

```sh
export PROJECT_ID=devops-toolkit-$(date +%Y%m%d%H%M%S)

gcloud projects create $PROJECT_ID

echo "https://console.cloud.google.com/marketplace/product/google/container.googleapis.com?project=$PROJECT_ID"
```

#### Open the URL and *ENABLE* the API

```sh
export SA_NAME=devops-toolkit

export SA="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

gcloud iam service-accounts \
    create $SA_NAME \
    --project $PROJECT_ID

export ROLE=roles/admin

gcloud projects add-iam-policy-binding \
    --role $ROLE $PROJECT_ID \
    --member serviceAccount:$SA

gcloud iam service-accounts keys \
    create gcp-creds.json \
    --project $PROJECT_ID \
    --iam-account $SA

kubectl --namespace crossplane-system \
    create secret generic gcp-creds \
    --from-file creds=./gcp-creds.json

kubectl crossplane install provider \
    crossplane/provider-gcp:v0.17.0
```

#### Wait for a few moments for the provider to be up-and-running
```sh
echo "apiVersion: gcp.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  projectID: $PROJECT_ID
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: gcp-creds
      key: creds" \
    | kubectl apply --filename -

kubectl apply --filename gcp.yaml
```

## Setup AWS 


#### Replace `[...]` with your access key ID`
```
export AWS_ACCESS_KEY_ID=[...]
```
#### Replace `[...]` with your secret access key
```sh
export AWS_SECRET_ACCESS_KEY=[...]

echo "[default]
aws_access_key_id = $AWS_ACCESS_KEY_ID
aws_secret_access_key = $AWS_SECRET_ACCESS_KEY
" | tee aws-creds.conf

kubectl --namespace crossplane-system \
    create secret generic aws-creds \
    --from-file creds=./aws-creds.conf

kubectl crossplane install provider \
    crossplane/provider-aws:v0.19.0
```
#### Wait for a few moments for the provider to be up-and-running

echo "apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-creds
      key: creds" \
    | kubectl apply --filename -

kubectl apply --filename aws.yaml


## Setup Azure 

az ad sp create-for-rbac \
    --sdk-auth \
    --role Owner \
    | tee azure-creds.json

export AZURE_CLIENT_ID=$(\
    cat azure-creds.json \
    | grep clientId \
    | cut -c 16-51)

export RW_ALL_APPS=1cda74f2-2616-4834-b122-5cb1b07f8a59

export RW_DIR_DATA=78c8a3c8-a07e-4b9e-af1b-b5ccab50a175

export AAD_GRAPH_API=00000002-0000-0000-c000-000000000000

az ad app permission add \
    --id "${AZURE_CLIENT_ID}" \
    --api ${AAD_GRAPH_API} \
    --api-permissions \
    ${RW_ALL_APPS}=Role \
    ${RW_DIR_DATA}=Role

az ad app permission grant \
    --id "${AZURE_CLIENT_ID}" \
    --api ${AAD_GRAPH_API} \
    --expires never

az ad app permission admin-consent \
    --id "${AZURE_CLIENT_ID}"

kubectl --namespace crossplane-system \
    create secret generic azure-creds \
    --from-file creds=./azure-creds.json

kubectl crossplane install provider \
    crossplane/provider-azure:v0.16.1

#### Wait for a few moments for the provider to be up-and-running

echo "apiVersion: azure.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: azure-creds
      key: creds" \
    | kubectl apply --filename -

kubectl apply --filename azure.yaml

###########################
# Creating infrastructure #
###########################

cat cluster.yaml

# Change `spec.compositionRef.name` to `cluster-gcp` or `cluster-azure` if not using AWS

kubectl apply --filename cluster.yaml

#######################
# Defining composites #
#######################

# It would be even better with Argo CD or Flux

cat definition.yaml

cat azure.yaml

cat gcp.yaml

cat aws.yaml

# If GCP
kubectl describe \
    composition cluster-google

# If Azure
kubectl describe \
    composition cluster-azure

# If AWS
kubectl describe \
    composition cluster-aws

kubectl explain \
    compositekubernetescluster \
    --recursive

#####################
# Resource statuses #
#####################

kubectl get compositekubernetesclusters

kubectl describe \
    compositekubernetescluster team-a

# If GCP
kubectl get gkeclusters,nodepools

# If Azure
kubectl get resourcegroups,aksclusters

# If AWS
kubectl get clusters,nodegroup,iamroles,iamrolepolicyattachments,vpcs,securitygroups,subnets,internetgateways,routetables

kubectl get compositekubernetesclusters

# Wait until it's up-and-running

####################################
# Accessing the new infrastructure #
####################################

kubectl --namespace team-a \
    get secret cluster \
    --output jsonpath="{.data.kubeconfig}" \
    | base64 -d \
    | tee kubeconfig.yaml

export KUBECONFIG=$PWD/kubeconfig.yaml

kubectl get nodes

kubectl get namespaces

unset KUBECONFIG

###########################
# Updating infrastructure #
###########################

# Open cluster.yaml in an editor
# Uncomment `spec.parameters.minNodeCount`

kubectl apply --filename cluster.yaml

export KUBECONFIG=$PWD/kubeconfig.yaml

kubectl get nodes


### Attach Cluster



[Follow steps here] (https://github.com/bkauf/attached)

#############################
# Destroying infrastructure #
#############################

unset KUBECONFIG

kubectl delete --filename cluster.yaml

kubectl get compositekubernetesclusters

kubectl get clusters

kubectl get clusters,nodegroup,iamroles,iamrolepolicyattachments,vpcs,securitygroups,subnets,internetgateways,routetables

# Wait until everything is removed

k3d cluster delete devops-toolkit
