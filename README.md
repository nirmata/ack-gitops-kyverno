# Demo for automated, secure, self-service EKS cluster provisioning using AWS Controllers for Kubernetes (ACK), ArgoCD and Kyverno

Secure self-service cluster provisioning workflow requires ACK EKS controller, Kyverno and ArgoCD to be deployed on a management cluster. Each of these components is preconfigured with appropriate permissions and to perform specific tasks
* ACK EKS Controller - Provisions the EKS cluster.
* Kyverno - Watches for resources and generates resources to initiate cluster creation and register the created cluster with ArgoCD.
* ArgoCD - Deploys the cluster resources to the management cluster and deploys the add-ons to the newly created tenant cluster.

## Setup

Create a management cluster on Amazon EKS and follow the instructions to install ACK, ArgoCD and Kyverno on the management cluster.

**Install and Configure ACK**

Install the ACK EKS controller by following the instructions [here](https://aws-controllers-k8s.github.io/community/docs/user-docs/install/). In order to use the controller, you have to configure IAM role for service account (IRSA) and give full access to the EKS service. Instructions to configure IRSA can be found [here](https://aws-controllers-k8s.github.io/community/docs/user-docs/irsa/#step-1-create-an-oidc-identity-provider-for-your-cluster).

Once the ACK EKS controller is configured, you can verify if it is configured by trying to create a cluster. Here are the sample YAMLs to create the cluster. Note: you will need to specify the cluster and node role ARNs.

**Install and Configure ArgoCD**

Next, install ArgoCD using the instructions to install the helm chart [here](https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd). 

ArgoCD will be used to install the add-ons once the cluster has been provisioned so it needs to be able to assume the same role that was used to create the cluster. You can find detailed instructions to install and configure ArgoCD on the EKS cluster [here](https://www.modulo2.nl/blog/argocd-on-aws-with-multiple-clusters).

Once ArgoCD is installed and verified, create the [AppSets for the cluster add-ons](appsets/). 

```console
kubectl create -f appsets/appset_kyverno.yaml -n argocd
kubectl create -f appsets/appset_kyverno_policies.yaml -n argocd
```

Optionally, if you want to automatically register your cluster to Nirmata Policy Manager (NPM), add the Nirmata Controller Registrator appset.

Note: You need to edit this appset YAML file and set the apiToken for your NPM account. To create the API token, go to Settings > Profile view, and click on Generate a new API Key. Once the API Key is generated use Copy API Key button to copy the API Key and paste it to the YAML file.

```console
kubectl create -f appsets/appset_nirmata_controller_chart.yaml -n argocd
```

You can verify the appsets:
```console
 kubectl get appsets -n argocd
```
In ArgoCD, change the resource tracking method to annotation so that it does not conflict with Kyverno as described [here](https://kyverno.io/docs/installation/#notes-for-argocd-users).

```console
kubectl edit cm -n argocd argocd-cm
```

```console
apiVersion: v1
data:
  application.resourceTrackingMethod: annotation
kind: ConfigMap
```

**Configure Kyverno**

Install Kyverno by following the instructions [here](https://kyverno.io/docs/installation/). 

Once Kyverno is installed, update the kyverno:generate clusterrole so that the ArgoCD Application resource can be created.

```console
- apiGroups:
  - argoproj.io
  resources:
  - applications
  verbs:
  - create
  - update
  - patch
  - delete
```

Now, apply the generate policies that will be used to create the ArgoCD application that creates the EKS cluster and register the cluster with ArgoCD.

```console
kubectl create -f policies/mgmt-cluster
```

The generate-application policy triggers the creation of an ArgoCD application that generates the resources to create the EKS cluster. This policy needs to be updated with cluster role ARN, node Role ARN, subnets and any other default configuration required for cluster and node group creation. 

The register-cluster policy registers newly created ACK clusters with ArgoCD so that the cluster add-ons can be deployed. 

**Add CreateCluster CRD**

Next we need a way for users to request clusters. This can be done in several different ways but here we will create a custom resource (CR) called CreateCluster in our management cluster. The custom resource definition can be found [here](cluster/crd). This CRD current allows a few fields such as name and desiredSize to be specified but it can be easily extended to include other fields that can be provided in the cluster creation request.

When the CreateCluster CR is created, the generate-application policy we created above will trigger and create an ArgoCD application that creates the Cluster, Nodegroup and Addon custom resources for EKS cluster creation.

**Create an EKS Cluster**

Once everything is in place, we can verify the setup. Apply the cluster creation request to your cluster.

```console
apiVersion: mgmt.nirmata.io/v1alpha1
kind: CreateCluster
metadata:
  name: ack-test-cluster
spec:
  name: ack-test-cluster
  desiredSize: 2
```

This should initiate creation of the ArgoCD application to deploy the cluster. Within a few seconds, you should be able to go to your Amazon EKS console and view that the cluster is being created. You can also verify the cluster creation in your management cluster using the commands:

```console
kubectl get cluster -n <namespace>
```

The cluster creation should take around 10-15 minutes. Once the cluster is created (in ACTIVE status), the register-cluster policy will trigger to register the newly created ACK cluster with ArgoCD so that the cluster add-ons can be deployed. This can be verified on the ArgoCD web console or by accessing the newly created cluster

If the Nirmata Controller Registrator appset has been configured, the cluster will show as registered in Nirmata Policy Manager.

The newly created cluster has Kyverno and the default policies installed. You can also install other required add-ons by creating corresponding ArgoCD appsets. This cluster is ready for use by your application teams!

