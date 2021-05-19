# Admin GitOps Repository

This repository contains cluster and namespace configuration.  It also contains configuration to deploy Operator instances such as Argo CD and Bitnami Sealed Secrets.

Only "cluster admins" should have access to commit to main.  This will help limit who can modify the cluster(s) and core cluster resources such as namespaces, resource quotas, and sealed secrets.

## Repository Layout

This repository has three main directories:
* **argocd**: This contains Argo CD resources (`AppProject` and `Project`) to configure two clusters - `non-prod` and `prod`
* **base**: This contains *shared* configuration that can be referenced from multiple *overlays*.  For example, the configuration of `argocd`, and `sealed-secrets` is the same between clusters, so this configuration is kept in the *base* folder and referenced from the appropriate *overlay* directory.
* **overlays**: This directory is split into `non-prod` and `prod` sub directories and contains the definition of each cluster.  In each *overlay* directory you will find a `kustomization.yaml` file that lists the resources (including bases) that make up the overall group of resources that are to be created (and managed) in a cluster.

## Prerequisites

To run this demo you will need:
* The "oc" command line tool.  This can be downloaded directly from OpenShift by clicking on the **?** icon near the top-right corner of the console, selecting "Command Line Tools" and downloading the binary for your operating system.
    * It is recommended to add this cli to your *path*.


## Getting Started

Reproducing this in your own cluster is mostly automated, but since there is sensitive information that needs to be stored in a git repository (GitHub and Quay credentials), there are a few manual tasks required in order to first encrypt your secrets.  

### 1. Fork the Repositories

Fork each of the demo repositories into your own GitHub account.  These are:
* [Contoso University DotNet App](https://github.com/pittar-sandbox/dotnet-contoso-app)
* [Admin GitOps Repository](https://github.com/pittar-sandbox/dotnet-contoso-gitops-admins)
* [Developers GitOps Repository](https://github.com/pittar-sandbox/dotnet-contoso-gitops-developers)

### 2. Update Argo CD Applications With Your Git Repositories

This part is tedious, but since you will need to update the *Applications* in the `argocd` directory of both the "admins" and "developers" repositories.  These files can be found in:

```
dotnet-contoso-gitops-admins/argocd
```
and
```
dotnet-contoso-gitops-developers/argocd
```

In the "developers" repository, you will also need to update URLs in the `pipeline` and `pipeline-run` files.

```
dotnet-contoso-gitops-developers/overlays/non-prod/01-namespaces/contoso-cicd/02-pipelines
```
and
```
dotnet-contoso-gitops-developers/pipeline-run/build-and-rollout-pipeline-run.yaml
```

These files end in `-app.yaml`.  You need to replace the `repoURL` with the link to your own git repository.

Commit and push these changes to your repositories.

### 3. Install the OpenShift GitOps Operator in Each Cluster

Log into your OpenShift clusters and install the "OpenShift GitOps" Operator.
1. From the "Administrator" view, select `Operators -> Operator Hub` from the left menu.
2. Search for **OpenShift GitOps**
3. Select this operator and click on **Install**, accepting all the defaults.

This will install the OpenShift GitOps operator, as well as a default instance of Argo CD in the `openshift-gitops` namespace.  This instance of Argo CD is meant for *cluster admins* to use to configure the cluster.

### 4. Create and Configure Namespaces with Argo CD

First, log into your "non-prod" OpenShift cluster using the `oc` cli as a *cluster-admin* user.

If you want to be able to access the default Argo CD instance without having to accept self-signed certificates, run the following command:

```
oc patch argocd argocd-cluster -n openshift-gitops --type=merge -p='{"spec":{"server":{"insecure":true,"route":{"enabled":true,"tls":{"insecureEdgeTerminationPolicy":"Redirect","termination":"edge"}}}}}'
```

Now for the real work!

Run the following to configure the demo projects/namespaces and some default access controls:

```
oc apply -k github.com/Gitvazonline/dotnet-contoso-gitops-admins/argocd/non-prod/01-namespaces
```

Once the namespaces are configured, install an instance of Argo CD for "Developers" and an instance of "Sealed Secrets" into your cluster with:

```
oc apply -k github.com/Gitvazonline/dotnet-contoso-gitops-admins/argocd/non-prod/02-operator-instances
```

**Note:** In OpenShift 4.6, the `argocd-for-developers-instance` will remain "OutOfSync".  This is fixed in Argo CD 2.0 / OpenShift 4.7.  For now, this is fine.  Everything will be running properly.

### 4. Configure Developer Namesapces

Now that the "Admins" have setup the cluster and created the namespaces for developers to use, it's time for the developers to do some things!

Run the following command to configure the developer namespaces and tooling:

```
oc apply -k github.com/Gitvazonline/dotnet-contoso-gitops-developers/argocd/non-prod/01-namespaces
```

This will do all the work!  Login to the "Developer" version of Argo CD, and you'll see the following Argo CD applications in various states of synchronization:
* cicd-tools
* contoso-cicd
* contoso-dev
* contoso-test



### 5. Trigger your first Pipeline Run

Your pipeline is ready and the build tools are configured.  Time to run a build!

Run the following command to kick off your first pipeline:

```
oc create -f https://raw.githubusercontent.com/Gitvazonline/dotnet-contoso-gitops-developers/main/pipeline-run/build-and-rollout-pipeline-run.yaml -n contoso-cicd
```

**Note:** Later, we can configure GitHub webhooks to automatically trigger pipeline runs when code is pushed to a repository.

In the OpenShift Developer console, change to the `contoso-cicd` project and click on "Pipelines" in the left menu.  You should see the `contoso-devops-pipeline` is running.  You can click on the pipeline run to watch the progress of the build.

When it is complete, you should have:
1. Fully functional applications in both `contoso-dev` and `contoso-test` namespace.
2. New `contoso` container images in your internal OpenShift repository.

