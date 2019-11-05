## Overview
These instructions are meant to be an easy to follow set of instructions for deploying Open-EBS storage on D2iQ Konvoy Kubernetes.  These instructions have been validated in the following environment:
* On-Prem Cluster
* Hyper-V 2016 Hypervisor VM's
* Centos 7.6 Control Plane & Kubelet nodes
* konvoy 1.2.3 kubernetes 1.15.4
* 3 x Control-Plane Nodes
* 6 x Kubelets (8cpu, 16GBram, 30GB root-drive, 4x60GB data-drives)
* Open-Ebs 1.3 with CSTOR storage provisioner

## Prerequisites
* Konvoy Kubernetes Cluster deployed without persistant storage and services requiring persistent storage disabled (see instructions).  See here for a sample [cluster.yaml-modified](https://raw.githubusercontent.com/dmennell/konvoyToolbox/master/install/konvoy-onPrem-openEbs/cluster.yaml-modified)
* 3+ x 55GB+ drives attached to each worker node (drives should not be partitioned/formatted)
* kubectl installed on client node and configured to access desired Kubernetes cluster
* Helm installed and configured on client node

## Install, Start, Enable, Validate iSCSI Client Software
This process needs to be completed on every Kubelet (non control-plane) node as Open EBS relies on the iSCSI Initiator and tools.  Run the following commands:
```
sudo yum install iscsi-initiator-utils -y
sudo systemctl start iscsid && sudo systemctl enable iscsid
```
Verify the name is configured:
```
cat /etc/iscsi/initiatorname.iscsi
```
Verify the service is running
```
systemctl status iscsid
```

## Deploy "OpenEBS Operator" on the Kubernetes Cluster
For Default Install, use Helm.
```
helm init
helm install --namespace openebs --name openebs stable/openebs --version 1.2.0
```

## Create Storage Pool Claim
First we need to get block devices, and then modify the YAML appropriately.

**Get Block Devices**

Run the following command.  
```
kubectl get blockdevices -n openebs
```
You will need the block device identifiers in the previous step.  Save the output to a text file or make sure it is available above in the terminal 

Create a [cstor-disk-pool.yaml](https://raw.githubusercontent.com/dmennell/konvoyToolbox/master/install/konvoy-onPrem-openEbs/cstor-disk-pool.yaml) file locally on your computer based on the one [HERE](https://raw.githubusercontent.com/dmennell/konvoyToolbox/master/install/konvoy-onPrem-openEbs/cstor-disk-pool.yaml)

**Modify & Deploy Storage Pool Claim**

Replace the `blockdevice` entries in yout local `cstor-disk-pool.yaml` file below with the entries from the the blockdevice list that you got froom the `kubectl get blockdevices -n openebs` command above, and deploy the YAML
```
kubectl apply -f cstor-disk-pool.yaml
```

## Create a Default Storage Class
Review the contents of [openebs-cstor-default.yaml](https://raw.githubusercontent.com/dmennell/konvoyToolbox/master/install/konvoy-onPrem-openEbs/openebs-cstor-default.yaml) and deploy the YAML
```
kubectl apply -f https://raw.githubusercontent.com/dmennell/konvoyToolbox/master/install/konvoy-onPrem-openEbs/openebs-cstor-default.yaml
```

## Deploy a Test-Pod to verify
First we will create the needed Persistant Volume Claim, and then we will deploy the pod to consume it.

**Create Persistent Volume Claim**

Review the contents of [pvc-test.yaml](https://raw.githubusercontent.com/dmennell/konvoyToolbox/master/install/konvoy-onPrem-openEbs/pvc-test.yaml) and deploy the YAML
```
kubectl apply -f https://raw.githubusercontent.com/dmennell/konvoyToolbox/master/install/konvoy-onPrem-openEbs/pvc-test.yaml
```

Verify Deployment
```
kubectl describe pvc pvc-test
```

**Deploy POD that uses PVC**

Review the contents of [pod-pv-test.yaml](https://raw.githubusercontent.com/dmennell/konvoyToolbox/master/install/konvoy-onPrem-openEbs/pod-pv-test.yaml) and deploy the YAML
```
kubectl apply -f https://raw.githubusercontent.com/dmennell/konvoyToolbox/master/install/konvoy-onPrem-openEbs/pod-pv-test.yaml
```

**Verify Deployment**
```
kubectl describe pod pod-pv-test
```

## Deploy Jenkins using Persistent Volumes from CSTOR

create a namespace for Jenkins
```
kubectl create ns jenkins
```
Review the contents of (jenkins.yaml)[https://raw.githubusercontent.com/dmennell/konvoyToolbox/master/install/konvoy-onPrem-openEbs/jenkins.yaml] and deploy the YAML
```
kubectl apply -f https://raw.githubusercontent.com/dmennell/konvoyToolbox/master/install/konvoy-onPrem-openEbs/jenkins.yaml
```

Verify that everything deployed properly and that it is exposed through Traefik
Once you have confirmed that everything is 
