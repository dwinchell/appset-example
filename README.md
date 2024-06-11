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
oc login --token=sha256~yourtokenfromhubui --server=https://api.cluster-pbntg.dynamic.redhatworkshops.io
oc config current-context # Copy context name
oc config rename-context default/api-cluster-pbntg-dynamic-redhatworkshops-io/admin hub

# Spoke 1
oc login --token=sha256~yourtokenfromspoke1ui --server=https://api.spoke1.example.com
oc config current-context # Copy context name
oc config rename-context default/api-spoke1.example.com/admin spoke1

# Spoke 2
oc login --token=sha256~yourtokenfromspoke2ui --server=https://api.spoke2.example.com
oc config current-context # Copy context name
oc config rename-context default/api-spoke2.example.com/admin spoke1

# Switch to Hub
oc config use-context hub

# Verify
oc config view
```

2. Install the OpenShift GitOps operator on the Hub. This will install an instance of ArgoCD. *Make sure you are using the hub context.*
```
oc config use-context hub

oc apply -f - << EOF
apiVersion: operators.coreos.com/v1alpha1
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
```

3. Wait for the OpenShift GitOps operator to install, and for the ArgoCD server to start.

This should take a few minutes. You can optionally use the command below to wait until the relevant Pod is running.
```
while true; do sleep 5; oc get po -n openshift-gitops -l app.kubernetes.io/name=cluster -o json | jq '.items[0].status.conditions[] | select(.type=="Ready").status=="True"' > /dev/null && break; echo -n .; done
```

4. Get the inital ArgoCD admin password, login, and reset the password.
```
export INITIAL_ARGOCD_PASSWORD=$(oc get secret -n openshift-gitops openshift-gitops-cluster -o json | jq '.data["admin.password"]' | sed 's/"//g' | base64 -d)
export NEW_ARGOCD_PASSWORD=$(tr -dc 'A-Za-z0-9!?%=' < /dev/urandom | head -c 30) # Generates a strong random password
argocd login ${ARGOCD_SERVER_URL} --username=admin --password=${INITIAL_ARGOCD_PASSWORD}
argocd account update-password --current-password=${INITIAL_ARGOCD_PASSWORD} --new-password=${NEW_ARGOCD_PASSWORD}
oc delete secret -n openshift-gitops openshift-gitops-cluster
```

5. Configure Hub to connect to and manage Spoke1 and Spoke2.

*WARNING: This will create a service account `argocd-manager` on the cluster referenced by context `spoke1` with full cluster level privileges.*

```
argocd cluster add -y spoke1
argocd cluster add -y spoke2
```

6. Create an ApplicationSet that will deploy an example application to Spoke 1 and Spoke 2.
```
oc apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: example
  namespace: openshift-gitops
spec:
  generators:
  - list:
      elements:
      - cluster: spoke1
        url: https://api.spoke1.example.com
      - cluster: spoke2
        url: https://api.spoke2.example.com
  goTemplate: true
  goTemplateOptions:
  - missingkey=error
  template:
    metadata:
      name: example-{{ .cluster }}
    spec:
      destination:
        namespace: example-{{ .cluster }}
        server: '{{ .url }}'
      project: default
      source:
        repoURL: https://github.com/validatedpatterns/multicloud-gitops.git
        path: "charts/all/hello-world/"
        targetRevision: HEAD
      syncPolicy:
        automated: {}
        syncOptions:
        - CreateNamespace=true
EOF
```

7. Verify the status of the ApplicationSet.
```
argocd --grpc-web appset list
```

ArgoCD shows a list of all ApplicationSets. The output should include a message indicating that ArgoCD has "Successfully generated parameters for all Applications".

8. Verify the status of the Applications.
```
argocd app list
```

9. Test that the workloads themselves are working. You can also visit the links in a browser.
```
curl hello-world-example-spoke1.apps.spoke1.example.com
curl hello-world-example-spoke2.apps.spoke2.example.com
```
