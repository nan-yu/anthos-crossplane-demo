**This Repo is not complete! Check back late October 2022 for the GA YAML**

Demo to control a crossplane cluster with ACM to provision GKE on GCP or EKS on AWS clusters with full lifecycle manageemnt of the infrastructure and k8s configurations. 

![Anthos and Crossplane for full lifecycle management of Cloud Infrastructure and K8s configuration](https://github.com/bkauf/anthos-crossplane-demo/blob/master/anthos_crossplane.jpg?raw=true)


Uncomment the file in crossplane-acm/clusters to create a GKE or EKS cluster

### Troubleshooting
```sh
kubectl get events -n default --sort-by={'lastTimestamp'}
```
