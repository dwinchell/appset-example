# Using OpenShift GitOps
An example of how to use OpenShift GitOps and ArgoCD ApplicationSets to manage workloads across multiple clusters.

# Introduction

As the number of OpenShift clusters in an enterprise grows, the effort of managing the configuration of each cluster also grows. This is true even if each cluster is managed individually using a GitOps approach.

There are a number of possible solutions to this challenge. One option is to adopt a hub and spoke architecture where a single hub cluster manages the configuration of the others, which are referred to as spokes. 

Sometimes this pattern is repeated between multi-cluster environments, which allows testing of changes to the hub before promoting them to production. You might have a set of development clusters managed by a development hub, and a set of production clusters managed by a production hub.

Choosing a hub and spoke architecture leaves open the question of how to implement the automation that allows the hub to control the spokes. One option is to deploy a single instance of ArgoCD to the hub and configure it to manage the spokes. This post walks through an example of that configuration.

OpenShift GitOps is Red Hat's solution for deploying supported instances of ArgoCD. It allows administrators to provision and configure ArgoCD using an operator and CRDs that are themselves GitOps-friendly. It also supports a multi-tenant model where multiple, dedicated instances of ArgoCD are automatically provision to support multiple tenant development teams while ensuring that each ArgoCD instance and team has only the needed permissions. This post uses OpenShift GitOps to deploy ArgoCD.

# Walkthrough

## Prerequisites

The instructions assume three small OpenShift clusters:
  - Hub - ArgoCD is deployed here. Does not host any workloads.
  - Spoke1 - Hosts example workloads.
  - Spoke2 - Hosts example workloads.

These can be small clusters. These instructions have been tested on single-node clusters with 16 cpu and 32 GiB of memory, but even smaller clusters would work. The same architecture works for Edge usecases using [Red Hat Build of MicroShift](https://access.redhat.com/documentation/en-us/red_hat_device_edge/4/html/overview/device-edge-overview) for the spoke clusters.

You will also need several CLI commands:
* `oc` - The OpenShift CLI
* `argocd` - the ArgoCD CLI
* `jq` - Not strictly required to perform the task, but these instructions use it to simplify certain steps.


## Instructions

1. Login to the OpenShift cluster that will be used as the Hub.

The example below assumes you are using a token for authentication, as if you were copy-pasting the login command from the web console. You can also login interactively with a username and password.

```
oc login --token=sha256~yourtokenfromhub --server=https://api.hub.example.com
```


2. Get the automatically generated name of the `oc` context that represents the connection to the Hub cluster.

Use the current-context command to display the context name.

```
oc current-context
```

Copy the context name for use in the next step.

!(Example Command Output. Copy paste the name of the context.)[/assets/oc-current-context-screenshot.png]

3. Rename the current context using the `oc rename-context` command. 

Copy the long, automatically generated context name from the previous output and paste it into this command. Set the new name to `hub`. This will make the rest of the commands easier.

**Important:** do not copy paste this command as-is. Fill in the generated context name from the output of the `oc current-context` command.

```
oc config rename-context default/api.hub.example.com/admin hub
```

4. Repeat steps 1-3 for the spoke clusters. For each cluster: login, get the context name, and rename it. Use `spoke1` and `spoke2` for the names.

```
# Spoke1
oc login --token=sha256~yourtokenfromspoke1 --server=https://api.spoke1.example.com
oc current-context
oc config rename-context default/api-spoke1.example.com/admin spoke1

# Spoke2
oc login --token=sha256~yourtokenfromspoke2 --server=https://api.spoke2.example.com
oc current-context
oc config rename-context default/api-spoke2.example.com/admin spoke2
```

5. Switch to the `hub` context for the next few commands. This will cause the commands you enter to execute on the Hub cluster. The use-context command is repeated below for clarity, but it is only necessary to enter it once each time you switch clusters.

```
oc config use-context hub
```

6. Install the OpenShift GitOps operator on the Hub. This will install an instance of ArgoCD. *Make sure you are using the hub context.*

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

This should only take a few minutes. You can optionally run the command below to monitor progress. Wait for a Pod with a name that starts with `openshift-gitops-server-` to appear and transition to status `Running`, then terminate the output with CTRL-c.
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

6. Create an ApplicationSet that will deploy an example application to Spoke1 and Spoke2.

Note: the commands below look up the URL for each cluster and save them to shell environment variables, which are then used in the `oc apply` command. By the time OpenShift receives the ApplicationSet definition, the shell has already filled in some of the values.

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

7. View the created ApplicationSet object.
```
oc get applicationset example -n openshift-gitops -o yaml
```

The output will look similar to this:

!(Example Output of the oc get applicationset command)[/assets/oc-get-applicationset-screenshot.png]

Notice that the URLs inside of the "generators:" section are filled in as constant values. Below that, in the "template:" section, they are referenced as variables.

This is because ArgoCD ApplicationSets have two parts - the generator(s) and the template.

A generator is a way of generating key-value pairs that are filled into the template. The output of the generator is a list of maps of parameters. In the example above we use the List generator. This is the simplest generator and it allows specifying values directly in the ApplicationSet definition. We generate two maps of parameters, one for each spoke cluster. The template will be instantiated twice, once with the "cluster" parmaeter set to "spoke1", and again with the "cluster" parameter set to "spoke2".

An ApplicationSet's template is used to create one or more Applications. The template is instantiated once for each map of parameters that is generated by the generator. Each instantiation creates one Application. The template language is simple, supporting only variable replacement. No if statements or functions are supported at the time of writing.

8. Verify the status of the ApplicationSet.
```
oc config use-context hub
argocd --grpc-web appset list
```

ArgoCD shows a list of all ApplicationSets. The output should include a message indicating that ArgoCD has "Successfully generated parameters for all Applications".

9. Verify the status of the Applications.
```
oc config use-context hub
argocd app list
```

Both applications should show a status of "Healthy". They might show a status of "Progressing" for up to one minute while they deploy.

10. Switch to the `spoke1` context for Spoke1 for the remaining commands. This will cause oc commands to execute on the Spoke1 cluster. The use-context command is repeated below for clarity, but it is only necessary to enter it once each time you switch clusters.
```
oc config use-context spoke1
```

11. Verify that the application Pod is running on Spoke1.
```
oc config use-context spoke1
oc get pod -n example-spoke1
```

The Pod should show a status of `Running`.

12. Optional - verify that the application returns data, using `curl`
```
oc config use-context spoke1
export SPOKE1_APP_URL=$(oc get route -n example-spoke1 hello-world -o json | jq -r .spec.host)
curl -k ${SPOKE1_APP_URL}
```

13. Opional - verify that you can browse to the application
Paste the URL printed by the commands below into your browser (or CTRL-click it).
```
oc config use-context spoke1
echo https://$(oc get route -n example-spoke1 hello-world -o json | jq -r .spec.host)
```

# Conclusion and Next Steps

Using OpenShift GitOps and ApplicationSets in a hub and spoke model reduces the toil and complexity of managing configuration and workloads across multiple clusters.

However, this technique alone does not address other challenges that arise when you have many clusters. Taking this model to the next level requires additional techniques and tooling.

Some immediate challenges that arise as the number of clusters grows include governance, observability, and the provisioning of the clusters themselves. [Red Hat Advanced Cluster Management for Kubernetes](https://www.redhat.com/en/technologies/management/advanced-cluster-management) can help address these challenges. It supports and compliments the model presented above, but a deep dive into how is worth a post of its own.

It can also be a challenge to secure a large number of clusters. [Red Hat Advanced Security for Kubernetes](https://www.redhat.com/en/technologies/cloud-computing/openshift/advanced-cluster-security-kubernetes) provides additional tooling that helps address that challenge.

It's also important to note that the coommands above presented a simple use case and already required a user to enter a number of shell commands. Doing this at scale requires a non-trivial amount of GitOps definitions and glue automation that you may not want to implement yourself. [Red Hat Edge Validated Patterns](https://www.redhat.com/en/products/edge/validated-patterns) address that sort of challenge by providing pre-defined, tested configurations that bring together the Red Hat portfolio and technology ecosystem. Not all patterns are specific to the Edge. In particular you may be interested in the [Multicloud GitOps Pattern](https://validatedpatterns.io/patterns/multicloud-gitops/), which uses a similar (though slightly different) architecture to the one described above.
