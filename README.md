# Komodor Helm RBAC

This repository is built to showcase how one can deploy a custom, self-maintained ServiceAccount and ClusterRole alongside the [komodor-agent] Helm Chart.

## What is the purpose of this repo?

By default, the above Helm Chart provides a mechanism to enable or disable Actions however this flag is global. Actions are updates from the Komodor UI to a Kubernetes cluster that can be triggered by Komodor users. The flag controlling this behavior is the `capabilities.actions`. As of today (Jan 22, 2025) the Helm Chart does not allow granular control over what type of Kubernetes resources a Komodor has user has permission to update. Note, this repo details how to control the komodor-agent's permissions inside a single Kubernetes cluster and does not have anything to do with configuring or controlling access from the [Komodor Platform](https://app.komodor.com). 

To reiterate, the goal of this repository is to deploy the [komodor-agent] Helm Chart, which onboards a cluster to the Komdor Platform, and provide a limited set of access and permission to the [komodor-agent]. For the purposes of this exercise, the limited set of access and permissions is to restrict the `komodor-agent's` ability to view all resoruces and make updates to only the Deployment resource type within our test cluster. Furthermore, we will limit the set of actions on Deployments to:

- scale
- rollback
- restart

## IMPORTANT

The Komodor Platform has rich [RBAC](https://help.komodor.com/hc/en-us/articles/22434021963154-RBAC) capabilities that allow an organization to create roles and policies that control user access. These roles and policies can be configured to grant a limited scope of access, such as permission to view all resources but only update resources of a particular Kubernetes type such the Deployments.

## How to Deploy

In order to achieve our desired goal, we will need to deploy the [komodor-agent] Helm Chart with a Helm `values.yaml` file that contains a few non-default configurations. Before we can deploy the Helm Chart we must ensure we deploy our custom ServiceAccount and ClusterRole.

### 1. Create the ServiceAccount and ClusterRole

Note, as our Helm Chart, ClusterRole, and ServiceAccount will all be located within the Komodor namespace and this namespace will not exist in a clean install, ensure you have created the komodor namespace if needed.

If the komodor namespace does not exist, go ahead and created it with the following command: `kubectl create namespace komodor`.

Utilizing `kubectl` and the `cluster-roles.yaml` file in the repo, we will create an explicit ServiceAccount and ClusterRole which will be used by the [komodor-agent]. After ensuring you have access to the cluster in which you want to onboard to Komodor, go ahead and create the resources via `kubectl apply -f cluster-roles.yaml`.

The output of applying the `cluster-roles.yaml` file should be:

```bash
clusterrole.rbac.authorization.k8s.io/komodor-agent created
clusterrolebinding.rbac.authorization.k8s.io/komodor-agent created
```

### 2. Deploying the [komdor-agent] Helm Chart

We are now ready to onboard our Kubernetes cluster to Komodor by deploying the [komodor-agent] Helm Chart. As we are going to make some changes to the default Helm Chart configuration, we will be supplying our own Helm `values.yaml` file. This file is located in this repo and is named `valuesForExternalClusterRole.yaml`.

Before executing the below commands, you will need to ensure you have the following:

- access to a [Komodor Platform account](https://app.komodor.com)
- a valid API_KEY obtained from your Komodor account
- a cluster name to identify the cluster from the Komodor Platform POV

Execute the following Helm command:

```bash
helm install --update komodor-agent komodorio/komodor-agent \
    --set apiKey={API_KEY_FROM_KOMODOR_PLATFORM} --set clusterName={CLUSTER_NAME_IN_KOMODOR_PLATFORM} \
    --namespace komodor -f valuesForExternalClusterRole.yaml
```

You should see the following output:

```bash
Release "komodor-agent" does not exist. Installing it now.
NAME: komodor-agent
LAST DEPLOYED: Wed Jan 22 13:00:39 2025
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing komodor-agent.

The Komodor Agent was installed in namespace: default

Visit our site at:
https://app.komodor.com

To learn more about Komodor please visit:
https://docs.komodor.com
```

The resources created in the above step can take seconds or minutes to fully create and become operational. To verify the success of your Helm Release, you can use the following command:

```bash
kubectl get all -n komodor
```

Once all resources have become fully operational, you can now return to the [Komodor Platform](https://app.komodor.com), log into your account and you should see your cluster is now sending information to the Komodor Platform.

## What did we accomplish?

In this exercise we have limited the `komodor-agent's` permissions, from a cluster level, to viewing all resources and being able to perform a subset of actions (scale, restart, rollback) for Deployments only. This means no matter what permissions are granted to a user in the Komodor Platform, those users can never have more permissions than viewing all resources and performing actions against Kubernetes Deployments.

If you attempt to perform an action against a different resource type, such as a Daemonset, you will see a Kubernetes RBAC error detailing your lack of permissions in the Komodor UI.

[komodor-agent]: https://github.com/komodorio/helm-charts/tree/master/charts/komodor-agent
