# Using OpenShift GitOps
An example of how to use OpenShift GitOps and ArgoCD ApplicationSets to manage workloads across multiple clusters.

# Introduction

As the number of OpenShift clusters in an enterprise grows, the effort of managing the configuration of each cluster also grows. This is true even if each cluster is managed individually using a GitOps approach.

There are a number of possible solutions to this challenge. One option is to adopt a hub and spoke architecture where a single hub cluster manages the configuration of the others, which are referred to as spokes. 

Sometimes this pattern is repeated between multi-cluster environments, which allows testing of changes to the hub before promoting them to production. You might have a set of development clusters managed by a development hub, and a set of production clusters managed by a production hub.

Choosing a hub and spoke architecture leaves open the question of how to implement the automation that allows the hub to control the spokes. One option is to deploy a single instance of ArgoCD to the hub and configure it to manage the spokes. This post walks through an example of that configuration.

OpenShift GitOps is Red Hat's solution for deploying supported instances of ArgoCD. It allows administrators to provision and configure ArgoCD using an operator and CRDs that are themselves GitOps-friendly. It also supports a multi-tenant model where multiple, dedicated instances of ArgoCD are automatically provision to support multiple tenant development teams while ensuring that each ArgoCD instance and team has only the needed permissions.

This post demonstrates how to use OpenShift GitOps to deploy ArgoCD and use it to manage workloads on multiple clusters in a hub and spoke model.

# Walkthrough

## Prerequisites

We will use three OpenShift clusters:
  - Hub - ArgoCD is deployed here. Does not host any workloads.
  - Spoke1 - Hosts example workloads.
  - Spoke2 - Hosts example workloads.

These can be small clusters. These instructions have been tested on single-node clusters with 16 cpu and 32 GiB of memory, but even smaller clusters would work. The same architecture works for Edge usecases using [Red Hat Build of MicroShift](https://access.redhat.com/documentation/en-us/red_hat_device_edge/4/html/overview/device-edge-overview) for the spoke clusters.

We will also use two CLI commands:
* `oc` - The OpenShift CLI
* `argocd` - the ArgoCD CLI

## Setting Up OpenShift GitOps

1. Login to the OpenShift cluster that will be used as the Hub.

The example below assumes you are using a token for authentication, as if you were copy-pasting the login command from the web console. You can also login interactively with a username and password.

```
oc login --token=sha256~yourtokenfromhub --server=https://api.hub.example.com
```


2. Get the automatically generated name of the `oc` context that represents the connection to the Hub cluster.

Use the current-context command to display the context name.

```
oc config current-context
```

![Example Command Output. Copy paste the name of the context.](/assets/oc-config-current-context_1.png)

Copy the context name for use in the next step.

3. Rename the current context using the `oc rename-context` command. 

Copy the long, automatically generated context name from the previous output and paste it into this command. Set the new name to `hub`. This will make the rest of the commands easier.

**Important:** do not copy paste this command as-is. Fill in the generated context name from the output of the `oc config current-context` command.

```
oc config rename-context default/api.hub.example.com/admin hub
```

4. Repeat steps 1-3 for the spoke clusters. For each cluster: login, get the context name, and rename it. Use `spoke1` and `spoke2` for the names.

```
# Spoke1
oc login --token=sha256~yourtokenfromspoke1 --server=https://api.spoke1.example.com
oc config current-context
oc config rename-context default/api-spoke1.example.com/admin spoke1

# Spoke2
oc login --token=sha256~yourtokenfromspoke2 --server=https://api.spoke2.example.com
oc config current-context
oc config rename-context default/api-spoke2.example.com/admin spoke2
```

5. Switch to the `hub` context for the next few commands. This will cause the commands you enter to execute on the Hub cluster.

```
oc config use-context hub
```

6. Install the OpenShift GitOps operator on the Hub. This will install an instance of ArgoCD. *Make sure you are using the hub context.*

```
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

7. Monitor the progress of the OpenShift GitOps installation.

This should only take a few minutes. You can optionally run the command below to monitor progress. 
```
oc get po -n openshift-gitops -w
```

8.  **Wait** for OpenShift GitOps to be installed, and for the operator to start an instance of ArgoCD.

When the output shows a Pod with a name that starts with `openshift-gitops-server-` to appear and transition to status `Running`, ArgoCD is ready.

When this has happened, terminate the command output with CTRL-c.

9. Get the inital ArgoCD admin password, login, and reset the password.
```
export INITIAL_ARGOCD_PASSWORD=$(oc get secret -n openshift-gitops openshift-gitops-cluster -o json | jq '.data["admin.password"]' | sed 's/"//g' | base64 -d)
export NEW_ARGOCD_PASSWORD=$(tr -dc 'A-Za-z0-9!?%=' < /dev/urandom | head -c 30) # Generates a strong random password
export ARGOCD_URL=$(oc get route -n openshift-gitops openshift-gitops-server -o json | jq -r .spec.host)

argocd login ${ARGOCD_URL} --username=admin --password=${INITIAL_ARGOCD_PASSWORD}

argocd account update-password --current-password=${INITIAL_ARGOCD_PASSWORD} --new-password=${NEW_ARGOCD_PASSWORD}
oc delete secret -n openshift-gitops openshift-gitops-cluster
```

6. Configure the Hub cluster to connect to and manage the Spoke1 and Spoke2 clusters.

*WARNING: This will create a service account `argocd-manager` on the cluster referenced by context `spoke1` with full cluster level privileges.*

```
argocd cluster add -y spoke1
argocd cluster add -y spoke2
```

If argocd commands display a warning that it failed to invoke a grpc calls, you can safely ignore this warning or add the --grpc-web flag to the commands.

We now have OpenShift GitOps installed on the Hub, and ArgoCD configured to connect to the Spokes.

## Creating a Basic ApplicationSet

7. Create an ApplicationSet that will deploy an example application to Spoke1 and Spoke2.

Note: The commands below look up the URL for each cluster and save them to shell environment variables, which are then used in the `oc apply` command. By the time OpenShift receives the ApplicationSet definition, the shell has already filled in some of the values. The next section will demonstrate how to avoid hardcoding cluster URLs in the ApplicationSet definition.

```
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

8. View the created ApplicationSet object.
```
oc get applicationset example -n openshift-gitops -o yaml
```

The output will include a lines similar to this:

![Example Output of the oc get applicationset command](/assets/oc-get-applicationset-2.png)

Notice that the URLs inside of the "generators:" section are filled in as constant values. Below that, in the "template:" section, they are referenced as variables.

We did it this way is because ArgoCD ApplicationSets have two parts - the generator(s) and the template.

A generator is a way of generating key-value pairs that are filled into the template. The output of the generator is a list of maps of parameters. In the example above we use the List generator. This is the simplest generator and it allows specifying values directly in the ApplicationSet definition. We generate two maps of parameters, one for each spoke cluster. The template will be instantiated twice, once with the "cluster" parmaeter set to "spoke1", and again with the "cluster" parameter set to "spoke2".

An ApplicationSet's template is used to create one or more Applications. The template is instantiated once for each map of parameters that is generated by the generator. Each instantiation creates one Application. The template language is simple, supporting only variable replacement. No if statements or functions are supported at the time of writing.

9. Verify the status of the ApplicationSet.
```
argocd --grpc-web appset list
```

ArgoCD shows a list of all ApplicationSets. The output should include a message indicating that ArgoCD has "Successfully generated parameters for all Applications".

10. Verify the status of the Applications.
```
argocd app list
```

Both applications should show a status of "Healthy". They might show a status of "Progressing" for up to one minute while they deploy.

11. Switch to the `spoke1` context for Spoke1 for the remaining commands. This will cause oc commands to execute on the Spoke1 cluster.
```
oc config use-context spoke1
```

12. Verify that the application Pod is running on Spoke1.
```
oc get pod -n example-spoke1
```

The Pod should show a status of `Running`.

13. Optional - verify that the application returns data, using `curl`
```
export SPOKE1_APP_URL=$(oc get route -n example-spoke1 hello-world -o json | jq -r .spec.host)
curl -k ${SPOKE1_APP_URL}
```

14. Opional - verify that you can browse to the application

Paste the URL printed by the commands below into your browser (or CTRL-click it).
```
echo http://$(oc get route -n example-spoke1 hello-world -o json | jq -r .spec.host)
```

## Using the Cluster Generator

As we saw in the previous section, we can specify key-value pairs using the simple List generator and then use them in a template. However, if values we want are the names and URLs of clusters, we can use better generator for this purpose - the Cluster generator.

15. Switch back to the hub context.

```
oc config use-context hub
```

16. Update the ApplicationSet to use the Cluster generator to fill in the cluster names and URLs.

```
oc apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: example
  namespace: openshift-gitops
spec:
  generators:
  - clusters: {}
  goTemplate: true
  goTemplateOptions:
  - missingkey=error
  template:
    metadata:
      name: example-{{ .name }}
    spec:
      destination:
        namespace: example-{{ .name }}
        server: '{{ .server }}'
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

17. Check the status of our apps after the change.

```
argocd app list
```

Now we have three applications, instead of two. What happened?

By default, the Cluster generator causes the template to be instantiated for every cluster configured in ArgoCD. That includes the cluster it is deployed to, so we get a third application. ArgoCD refers to that cluster as "in-cluster", so based on our template the name of the appliction is "example-in-cluster".

But why is the new application out of sync?

18. Investigate the new "example-in-cluster" application.

```
argocd app get example-in-cluster
```

The output is long, but we have an error message similar to, `SyncError  Failed sync attempt to d97644d7d1bb261837aea069a7fb73c3b7881b71: one or more objects failed to apply, reason: routes.route.openshift.io is forbidden: User "system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller" cannot create resource "routes" ...`

ArgoCD is attempting to create the resources for our example application, but it fails because the serviceaccount that it is using does not have the required permissions. The default instance of ArgoCD created by the OpenShift GitOps operator does not have permission to do deploy our application.

We could give ArgoCD permission to deploy the application to the Hub, but let's assume for now that we only want to deploy the application to the Spokes.

19. View the list of ArgoCD cluster secrets

When we ran the `argocd cluster add` commands, that created two kubernetes Secret objects in the Hub cluster. These are called "cluster secrets".

```
oc get secret -l argocd.argoproj.io/secret-type=cluster -n openshift-gitops
```

We can see that the Secrets are marked with the label, `argocd.argoproj.io/secret-type=cluster`. ArgoCD uses this label to distinguish cluster Secrets.

There is no cluster secret for the in-cluster cluster.

20. Label the cluster secrets for the two spokes.

```
oc label secret -l argocd.argoproj.io/secret-type=cluster -n openshift-gitops example.company.com/cluster-role=spoke
```

21. View the updated labels.

```
oc get secret -l argocd.argoproj.io/secret-type=cluster -n openshift-gitops
```

19. Configure the cluster generator to target only the spoke clusters.

Let's tell the Cluster generator that we only want to instantiate the template for cluster secrets that have the label "example.company.com/cluster-role" with a value of "spoke".

```
oc apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: example
  namespace: openshift-gitops
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          example.company.com/cluster-role: spoke
  goTemplate: true
  goTemplateOptions:
  - missingkey=error
  template:
    metadata:
      name: example-{{ .name }}
    spec:
      destination:
        namespace: example-{{ .name }}
        server: '{{ .server }}'
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

23. Verify that there are only two applications now.

```
argocd app list
```

As expected, the Cluster generator no longer selects the "in-cluster" cluster, and we are back down to two applications.

# Conclusion and Next Steps

Using OpenShift GitOps in a hub and spoke model reduces the toil and complexity of managing configuration and workloads across multiple clusters. We can use ArgoCD ApplicationSets, templates and generators to simplify the generation of Application objects for many clusters.

However, these techniques alone do not address other challenges that arise when you have many clusters. Taking this model to the next level requires additional techniques and tooling.

Some immediate challenges that arise as the number of clusters grows include governance, observability, and the provisioning of the clusters themselves. [Red Hat Advanced Cluster Management for Kubernetes](https://www.redhat.com/en/technologies/management/advanced-cluster-management) can help address these challenges. It supports and compliments the model presented above, but a deep dive into how is worth a post of its own.

It can also be a challenge to secure a large number of clusters. [Red Hat Advanced Security for Kubernetes](https://www.redhat.com/en/technologies/cloud-computing/openshift/advanced-cluster-security-kubernetes) provides additional tooling that helps address that challenge.

It's also important to note that the coommands above presented a simple use case and already required a user to enter a number of shell commands. Doing this at scale requires a non-trivial amount of GitOps definitions and glue automation that you may not want to implement yourself. [Red Hat Edge Validated Patterns](https://www.redhat.com/en/products/edge/validated-patterns) address that sort of challenge by providing pre-defined, tested configurations that bring together the Red Hat portfolio and technology ecosystem. Not all patterns are specific to the Edge. In particular you may be interested in the [Multicloud GitOps Pattern](https://validatedpatterns.io/patterns/multicloud-gitops/), which uses a broadly similar approach to the one described above.
