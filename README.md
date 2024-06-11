# ArgoCD AppSet Example
An example of how to use ArgoCD ApplicationSets to manage workloads across multiple clusters.

# Architecture

This example assumes a hub and spoke architecture. 

ArgoCD runs only on the hub cluster. It is used to manage workloads on the spoke clusters. ArgoCD ApplicationSets are used to create Applications that define which workloads ArgoCD should deploy to each cluster.

The instructions assume you are running on OpenShift, but the architecture is relevant to any Kubernetes environment and most of the commands should work as-is in other Kubernetes environments. The OpenShift GitOps operator is used to install ArgoCD, but you could install upstream ArgoCD using another method without affecting the rest of the instructions.

# Prerequisites

The instructions assume three small OpenShift clusters:
  - Hub - ArgoCD is deployed here. Does not host any workloads.
  - Spoke1 - Hosts example workloads.
  - Spoke2 - Hosts example workloads.

These can be small clusters. Single node clusters with 16 cpu and 32 GiB are big enough.

If you only have 1 or 2 clusters, you can substitute cluster URLs and it should work fine with no other changes.

You will also need the `oc` and `argocd` CLIs.

# Setup
1. Using `oc`, Login to all of the clusters as a user with the cluster-admin role. Rename each oc context to make it easier to switch between clusters later.
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

# Switch to Hub
oc config use-context hub

# Verify
oc config view

2. Install the OpenShift GitOps operator on the Hub. This will install an instance of ArgoCD. *Make sure you are using the hub context.*
```
oc config use-context hub

oc apply -f - << EOF
> apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: latest
  installPlanApproval: Automatic
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Wait for ArgoCD to start. You can also just watch the console.
while true; do sleep 5; oc get po -n openshift-gitops -l app.kubernetes.io/name=cluster -o json | jq '.items[0].status.conditions[] | select(.type=="Ready").status=="True"' > /dev/null && break; echo -n .; done
```

3. Get the inital ArgoCD admin password, login, and reset the password.
```
export INITIAL_ARGOCD_PASSWORD=$(oc get secret -n openshift-gitops openshift-gitops-cluster -o json | jq '.data["admin.password"]' | sed 's/"//g' | base64 -d)
export NEW_ARGOCD_PASSWORD=$(tr -dc 'A-Za-z0-9!?%=' < /dev/urandom | head -c 30) # Generates a strong random password
argocd login ${ARGOCD_SERVER_URL} --username=admin --password=${INITIAL_ARGOCD_PASSWORD}
argocd account update-password --current-password=${INITIAL_ARGOCD_PASSWORD} --new-password=${NEW_ARGOCD_PASSWORD}
oc delete secret -n openshift-gitops openshift-gitops-cluster
```

4. Configure Hub to connect to and manage Spoke1 and Spoke2.

*WARNING: This will create a service account `argocd-manager` on the cluster referenced by context `spoke1` with full cluster level privileges.*

```
argocd cluster add -y spoke1
argocd cluster add -y spoke2
```
