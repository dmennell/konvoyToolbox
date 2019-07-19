these instructions are what I have used to deploy Konvoy 0.4.0 on Hyper-V based VM's with local stateful storage.  3 x Master node contains 8vcpu, 16gb RAM, and 80GB root hard drive. 6 x Worker nodes have 8vcpu, 16gb RAM, 80GB root hard drive, and 180gb hdd each at /dev/sdb, /dev/sdc. and /dev/sdd.

#Create Inventory.yaml That Looks Like This
Uopdate the IP Addresses, the Private Key, the Ansible User.
```
control-plane:
  hosts:
    192.168.2.151:
      ansible_host: 192.168.2.151
    192.168.2.152:
      ansible_host: 192.168.2.152
    192.168.2.153:
      ansible_host: 192.168.2.153

node:
  hosts:
    192.168.2.154:
      ansible_host: 192.168.2.154
      node_pool: node
    192.168.2.155:
      ansible_host: 192.168.2.155
      node_pool: node
    192.168.2.156:
      ansible_host: 192.168.2.156
      node_pool: node
    192.168.2.157:
      ansible_host: 192.168.2.157
      node_pool: node
    192.168.2.158:
      ansible_host: 192.168.2.158
      node_pool: node
    192.168.2.159:
      ansible_host: 192.168.2.159
      node_pool: node

all:
  vars:
    ansible_port: 22
    ansible_ssh_private_key_file: "dcm-konvoy-key"
    ansible_user: "dcmennell"
    control_plane_endpoint: ""
    order: sorted
```

#Set Up the Drives
This step assumes that the 3 additional drives are un-initialized, and unformatted.  It also assumes that the drives are available at /dev/sdb, /dev/sdc, and /dev/sdd.  You will first need to ssh into each worker node

##Create the File System on each drive
```
sudo mkfs.ext4 /dev/sdb
#confirm with a y

sudo mkfs.ext4 /dev/sdc
#confirm with a y

sudo mkfs.ext4 /dev/sdd
#confirm with a y
```

##Create the /mnt/disks directory
```
sudo mkdir /mnt/disks
```

##Mount Drives & Modify "fstab" (1 at a time)
Do this fo each drive /dev/sdb, /dev/sdc, /dev/sdd
First you will need to ssh into the nodes with the appropriate key
```
#Mount /dev/sdb
DISK_UUID=$(sudo blkid -s UUID -o value /dev/sdb)
sudo mkdir /mnt/disks/$DISK_UUID
sudo mount -t ext4 /dev/sdb /mnt/disks/$DISK_UUID
echo UUID=`sudo blkid -s UUID -o value /dev/sdb` /mnt/disks/$DISK_UUID ext4 defaults 0 2 | sudo tee -a /etc/fstab

#Mount /dev/sdc
DISK_UUID=$(sudo blkid -s UUID -o value /dev/sdc)
sudo mkdir /mnt/disks/$DISK_UUID
sudo mount -t ext4 /dev/sdc /mnt/disks/$DISK_UUID
echo UUID=`sudo blkid -s UUID -o value /dev/sdc` /mnt/disks/$DISK_UUID ext4 defaults 0 2 | sudo tee -a /etc/fstab

#Mount /dev/sdd
DISK_UUID=$(sudo blkid -s UUID -o value /dev/sdd)
sudo mkdir /mnt/disks/$DISK_UUID
sudo mount -t ext4 /dev/sdd /mnt/disks/$DISK_UUID
echo UUID=`sudo blkid -s UUID -o value /dev/sdd` /mnt/disks/$DISK_UUID ext4 defaults 0 2 | sudo tee -a /etc/fstab
```

##Reboot
```
sudo reboot
```

#Set Skip AWS and Initialize
```
export SKIP_AWS=true
./konvoy init --provisioner=none [--cluster-name honey-badger]
```

#Modify the cluster.yaml so it looks something like this
```
---
kind: ClusterConfiguration
apiVersion: konvoy.mesosphere.io/v1alpha1
metadata:
  name: konvoy_v0.4.0
  creationTimestamp: "2019-07-17T14:16:46.2148896Z"
spec:
  kubernetes:
    version: 1.15.0
    controlPlane:
      controlPlaneEndpointOverride: "192.168.2.180:6443"
      keepalived:
        enabled: true
        interface: eth0
    networking:
      podSubnet: 172.16.0.0/16
      serviceSubnet: 10.0.0.0/18
      httpProxy: ""
      httpsProxy: ""
    cloudProvider:
      provider: none
    podSecurityPolicy:
      enabled: false
    preflightChecks:
      errorsToIgnore: ""
  containerNetworking:
    calico:
      version: v3.8.0
  containerRuntime:
    containerd:
      version: 1.2.5
      configData:
        data: ""
        replace: false
  nodePools:
  - name: worker
  addons:
    configVersion: v0.0.29
    addonsList:
    - name: elasticsearchexporter
      enabled: true
    - name: helm
      enabled: true
    - name: opsportal
      enabled: true
    - name: prometheusadapter
      enabled: true
    - name: velero
      enabled: true
    - name: minio
      enabled: true
    - name: dashboard
      enabled: true
    - name: dex
      enabled: true
    - name: dex-k8s-authenticator
      enabled: true
    - name: elasticsearch
      enabled: true
    - name: fluentbit
      enabled: true
    - name: kibana
      enabled: true
    - name: localvolumeprovisioner
      enabled: true
    - name: metallb
      enabled: true
      values: |-
        configInline:
          address-pools:
          - name: default
            protocol: layer2
            addresses:
            - 192.168.2.181-192.168.2.199
    - name: prometheus
      enabled: true
    - name: traefik
      enabled: true
    - name: kommander
      enabled: true
    - name: traefik-forward-auth
      enabled: true
  version: v0.4.0
```

#Run Preflight
```
./konvoy check preflight
```

#Build the Kluster
```
./konvoy up
```

There is a good chance that it will timeout at the beginning of the addons section. with an error like this:
```
STAGE [Deploying Enabled Addons]

Error: failed to deploy the cluster: error generating workflow from Addon list: failed to create workflow for https://github.com/mesosphere/kubeaddons-configs@v0.0.29: failed to deploy addon CRD opsportal; error: deployConfig failed: exit status 1 [stderr: error: unable to recognize "STDIN": no matches for kind "Addon" in version "kubeaddons.mesosphere.io/v1alpha1"
]
exit status 1
```
If so, run `./konvoy up` again and it will cycle through the already completed steps and proceed to the addon deployment

#Set KubeConfig File
use the full path so that if you change directories, it will still work.
```
export KUBECONFIG=/Users/dcmennell/clusters/konvoy/badger/konvoy_v0.4.0/admin.conf
```

#Expose Kommander
```
kubectl expose deployment kommander-deployment -n=kommander --type=LoadBalancer --name=k7r-dash
```
Find the address the service was exposed over:
```
kubectl get services -n=kommander
```

