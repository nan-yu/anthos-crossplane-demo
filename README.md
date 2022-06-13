Demo to control a crossplane cluster with ACM to provision GKE on GCP or EKS on AWS clusters

Uncomment the file in crossplane-acm/clusters to create a GKE or EKS cluster

### Troubleshooting
```sh
kubectl get events -n <namespace> --sort-by={'lastTimestamp'}
```