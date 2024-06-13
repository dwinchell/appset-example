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

You will also need several CLI commands:
* `oc` - The OpenShift CLI
* `argocd` - the ArgoCD CLI
* `jq` - Not strictly required to perform the task, but these instructions use it to simplify certain tasks.

# Setup
1. Using `oc`, Login to all of the clusters as a user with the cluster-admin role. Rename each oc context to make it easier to switch between clusters later.
```
# Login
oc login --token=sha256~yourtokenfromhubui --server=https://api.hub.example.com
oc login --token=sha256~yourtokenfromspoke1ui --server=https://api.spoke1.example.com
oc login --token=sha256~yourtokenfromspoke2ui --server=https://api.spoke2.example.com

# View the names of the contexts
oc config view

# Rename each context for easier management later
oc config rename-context default/api.hub.example.com/admin hub
oc config rename-context default/api-spoke1.example.com/admin spoke1
oc config rename-context default/api-spoke2.example.com/admin spoke2

# Switch to Hub
oc config use-context hub
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

3. *Wait* for the OpenShift GitOps operator to install, and for the ArgoCD server to start.

This should take a few minutes. You can optionally run the command below to monitor progress. Wait for a Pod with a name that starts with `openshift-gitops-server-` to appear and transition to status `Running`, then terminate the output with CTRL-c.
```
oc config use-context hub
oc get po -n openshift-gitops -w
```

4. Get the inital ArgoCD admin password, login, and reset the password.
```
oc config use-context hub

export INITIAL_ARGOCD_PASSWORD=$(oc get secret -n openshift-gitops openshift-gitops-cluster -o json | jq '.data["admin.password"]' | sed 's/"//g' | base64 -d)
export NEW_ARGOCD_PASSWORD=$(tr -dc 'A-Za-z0-9!?%=' < /dev/urandom | head -c 30) # Generates a strong random password
export ARGOCD_URL=$(oc get route -n openshift-gitops openshift-gitops-server -o json | jq -r .spec.host)

argocd login ${ARGOCD_URL} --username=admin --password=${INITIAL_ARGOCD_PASSWORD}

argocd account update-password --current-password=${INITIAL_ARGOCD_PASSWORD} --new-password=${NEW_ARGOCD_PASSWORD}
oc delete secret -n openshift-gitops openshift-gitops-cluster
```

5. Configure the Hub cluster to connect to and manage the Spoke1 and Spoke2 clusters.

*WARNING: This will create a service account `argocd-manager` on the cluster referenced by context `spoke1` with full cluster level privileges.*

```
oc config use-context hub
argocd cluster add -y spoke1
argocd cluster add -y spoke2
```

6. Create an ApplicationSet that will deploy an example application to Spoke 1 and Spoke 2.
```
oc config use-context hub
export SPOKE1_CLUSTER_URL=$(argocd cluster list -o json | jq -r '.[] | select(.name=="spoke1").server')
export SPOKE2_CLUSTER_URL=$(argocd cluster list -o json | jq -r '.[] | select(.name=="spoke2").server')

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
        url: ${SPOKE1_CLUSTER_URL}
      - cluster: spoke2
        url: ${SPOKE2_CLUSTER_URL}
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
oc config use-context hub
argocd --grpc-web appset list
```

ArgoCD shows a list of all ApplicationSets. The output should include a message indicating that ArgoCD has "Successfully generated parameters for all Applications".

8. Verify the status of the Applications.
```
oc config use-context hub
argocd app list
```

Both applications should show a status of "Healthy". They might show a status of "Progressing" for up to one minute while they deploy.

9. Verify that the Pod is running on Spoke1

```
oc config use-context spoke1
oc get pod -n example-spoke1
```

The Pod should show a status of `Running`.

10. Optional - verify that the application returns data, using `curl`
```
oc config use-context spoke1
export SPOKE1_APP_URL=$(oc get route -n example-spoke1 hello-world -o json | jq -r .spec.host)
curl -k ${SPOKE1_APP_URL}
```

11. Opional - verify that you can browse to the application
Paste the URL printed by the commands below into your browser (or CTRL-click it).
```
oc config use-context spoke1
echo https://$(oc get route -n example-spoke1 hello-world -o json | jq -r .spec.host)
```
