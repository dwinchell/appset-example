# ArgoCD AppSet Example
An example of how to use ArgoCD ApplicationSets to manage workloads across multiple clusters.

# Architecture

This example assumes a hub and spoke architecture. 

ArgoCD runs only on the hub cluster. It is used to manage workloads on the spoke clusters. ArgoCD ApplicationSets are used to create Applications that define which workloads ArgoCD should deploy to each cluster.

The instructions assume you are running on OpenShift, but the architecture is relevant to any Kubernetes environment and most of the commands should work as-is in other Kubernetes environments. The OpenShift GitOps operator is used to install ArgoCD, but you could install upstream ArgoCD using another method without affecting the rest of the instructions.

# Prerequisites

The instructions assume three OpenShift clusters:
  - One as a hub that will be used to deploy ArgoCD.
  - Two as spokes that will be used to deploy only workloads.

If you only have 1 or 2 clusters, you can substitute cluster URLs and it should work fine with no other changes.

You will also need the `oc` and `argocd` CLIs.

# Setup
1. Login to all of the clusters with oc. Rename each kubectl context to make it easier to switch between clusters later.
```
# Hub
oc login --token=sha256~A2w0Et-SaQqg64pTZei34FJHn6CKHc_kpR67kSrQBsI --server=https://api.cluster-pbntg.dynamic.redhatworkshops.io:6443
oc config current-context
oc config rename-context default/api-cluster-pbntg-dynamic-redhatworkshops-io:6443/admin hub

# Spoke 1
oc login --token=sha256~M_Ub8LhyzctSTZcsu-A8wcq29ttxQWQUoCYXdlZp57k --server=https://api.cluster-9vkr6.dynamic.redhatworkshops.io:6443
oc config current-context
oc config rename-context default/api-cluster-9vkr6-dynamic-redhatworkshops-io:6443/admin spoke1

# Spoke 2
oc login --token=sha256~tEizXc-sIy8P3bSVDhRAAl93sWg4ZgsNHBiC-Ow4nOI --server=https://api.cluster-zscjv.dynamic.redhatworkshops.io:6443
oc config current-context

# Verify
oc config use-context hub

# Switch to Hub
oc config view

2. Install the OpenShift GitOps operator on the Hub. This will install an instance of ArgoCD.
```
oc apply -f openshift-gitops-subscription.yaml
```
