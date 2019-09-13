## Overview
These instructions are meant to be an easy to follow set of instructions for deploying Open-EBS storage on D2iQ Konvoy Kubernetes.  These instructions have been validated in the following environment:
* On-Prem Cluster
* Hyper-V 2016 Hypervisor VM's
* Centos 7.6 Control Plane & Kubelet nodes
* konvoy 1.1.5 kubernetes 1.15.3
* 3 x Control-Plane Nodes
* 9 x Kubelets (8cpu, 16GBram, 30GB root-drive, 4x60GB data-drives)
* Open-Ebs 1.2 with CSTOR storage provisioner


## Prerequisites
* Konvoy Kubernetes Cluster deployed without persistant storage and services requiring persistent storage disabled (see instructions)
* 3+ x 55GB+ drives attached to each worker node (drives should not be partitioned/formatted)
* kubectl installed on client node and configured to access desired Kubernetes cluster
* Helm installed and configured on client node

## Install, Start, Enable, Validate iSCSI Client Softweare
This process needs to be completed on every Kubelet (non control-plane) node as Open EBS relies on the iSCSI Initiator and tools
```
sudo yum install iscsi-initiator-utils -y
sudo systemctl enable iscsid
sudo systemctl start iscsid
```
Validate iSCSI Installation:

Is Innitiator Name Configured?
```
cat /etc/iscsi/initiatorname.iscsi
```
Is Service Running?
```
systemctl status iscsid
```

## Deploy "OpenEBS Operator" on the Kubernetes Cluster

For Default Install, use Helm
```
helm init
helm install --namespace openebs --name openebs stable/openebs --version 1.2.0
```

## Create Storage Pool Claim
First we need to get block devices, and then modify the yaml appropriately.

**Get Block Devices**
you will need the Block Device identifiers in th

```
kubectl get blockdevices -n openebs
```
save the output or make sure it is available to copy/paste into the 

**Modify & Deploy Storage Pool Claim**

replace the `blockDeviceList` entries below with the entries from the the command above. and
deploy the YAML
```
kubectl apply -f cstor-disk-pool.yaml
```

## Create a Default Storage Class
review the contents of `openebs-cstor-default.yaml` and deploy the YAML
```
kubectl apply -f openebs-cstor-default.yaml
```

## Deploy a Test-Pod to verify
First we will create the needed Persistant Volume Claim, and then we will deploy the pod to consume it.

**Create Persistent Volume Claim**

review `pvc-test.yaml` and deploy the YAML
```
kubectl apply -f pvc-test.yaml
```

Verify Deployment

```
kubectl describe pvc pvc-test
```

**Deploy POD that uses PVC**

review `pod-pv-test.yaml` and deploy the YAML
```
kubectl apply -f pod-pv-test.yaml
```

**Verify Deployment**
```
kubectl describe pod pod-pv-test
```

Deploy Jenkins using Persistent Volumes from CSTOR

create a namespace for Jenkins
```
kubectl create ns jenkins
```
review `jenkins.yaml` and deploy the YAML
```
kubectl apply -f jenkins.yaml
```

Verify that everything deployed properly and that it is exposed through Traefik

