# vSphere VM Creation Supply Chain
This is an example for a cartographer supply chain that is atypical.  
A typical supply chain deploys an application to kubernetes where as this supply chain deploys a VM on vSphere.

# Overview
This example illustrates how an App Operator group could set up a software  
supply chain such that source code gets continuously built using the best  
practices from [Packer] and deployed to vSphere as a VM using [Terraform].

```

  Git Source --> VM Template --> VM

```  
# Prerequisites

1. Kubernetes v1.19+
2. vSphere 6.7+
3. [Carvel](https://carvel.dev) tools for templating and groups of Kubernetes objects to the api
   server

  - [ytt](https://carvel.dev/ytt): templating the credentials
  - [kapp](https://carvel.dev/kapp): submitting objects to Kubernetes
  - [kapp controller](https://carvel.dev/kapp-controller): Optional but will be very helpful for installation of all prerequisites in the cluster
4. [Cartographer](https://cartographer.sh)
5. [Flux Source Controller](https://fluxcd.io/docs/components/source)
5. [Tekton](https://tekton.dev)
6. [Terraform Controller](https://github.com/oam-dev/terraform-controller)
7. [Tanzu CLI](https://tanzucommunityedition.io): this is not required but is used bellow for installation of prerequisites and for ease of management
8. [Helm v3 CLI](https://helm.sh/docs/intro/install/)

# Prerequisites Installation
The easiest way to install all the needed prerequisites is to use the [TAP OSS project](https://github.com/vrabbi/tap-oss).  
To install just the needed parts you can run the following set of commands:  
```bash
# Preperation
kubectl create ns tap-oss
tanzu package repository add tap-oss -n tap-oss --url ghcr.io/vrabbi/tap-oss-repo:0.2.4

# Installation
tanzu package install cert-manager -p cert-manager.tap.oss -v 1.6.1 -n tap-oss
tanzu package install cartographer -p cartographer.tap.oss -v 0.2.1 -n tap-oss
tanzu package install flux-source-controller -p flux-source-controller.tap.oss -v 0.21.1 -n tap-oss
tanzu package install tekton-pipelines -p tekton.tap.oss -v 0.32.1 -n tap-oss
helm repo add kubevela-addons https://charts.kubevela.net/addons
helm upgrade --install terraform-controller -n terraform --create-namespace kubevela-addons/terraform-controller --set backend.namespace=terraform
```  

# Installing the example supply chain
1. Update the values file in this repo to include your environment specific values for example:
```bash
#@data/values
---
#! VM Folder in vSphere inventory to create the Template and VM in
vm_folder: demo-folder
#! The vSphere Portgroup name to use for the template creation and VM deployment (Must have DHCP enabled on the network)
vm_network: demo-tf-vms
#! The vSphere Cluster to deploy to
cluster_name: Demo-Cluster
#! The VM name to deploy - (VM name will be suffixed with the deployment time stamp automatically)
vm_name: web-01
#! Datastore name in vSphere to deploy to
datastore_name: vsanDatastore
#! The vSphere Datacenter to deploy to
datacenter_name: Demo-Datacenter
#! vSphere user name to use for packer and Terraform operations
vcenter_username: administrator@vsphere.local
#! Password for the vSphere user to use for packer and Terraform operations
vcenter_password: Bpovtmg1!
#! SSH username to create on the new VM
vm_user: demo
#! SSH Password for the new user being created on the VM
vm_password: Aa123456
#! FQDN or IP of the vCenter server this supply chain will target
vcenter_fqdn: demo-vc-01.terasky.demo
#! Boolean value to allow or deny insecure vSphere access aka no cert validation
vcenter_insecure: true
```
2. Use YTT and Kapp to deploy the supply chain objects to the cluster using the values file you just updated:
```bash
ytt -f ./templates/ -f values.yaml | kapp deploy -a vsphere-vm-carto-sc -f - -y
```  
3. Deploy an example workload
```bash
kubectl apply -f workload.yaml
```  
4. Once all steps have been completed you can view to object tree that was created for our workload using a CLI tool like [kubectl lineage](https://github.com/tohjustin/kube-lineage):
```
# Option 1 - Tree view
kubectl lineage workload/demo-app --exclude-types events

# Option 2 - Table view
kubectl lineage workload/demo-app --exclude-types events --output=split
```  
5. To get the IP of the deployed VM from kubernetes you can do the following:
```bash
kubectl get configurations.terraform.core.oam.dev demo-app -o jsonpath='{.status.apply.outputs.ip.value}'
```
