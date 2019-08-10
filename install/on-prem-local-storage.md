# Konvoy Install - OnPrem with Local Storage

these instructions are what I have used to deploy Konvoy on Hyper-V based VM's with local stateful storage.  3 x Master node contains 8vcpu, 16gb RAM, and 80GB root hard drive. 6 x Worker nodes have 8vcpu, 16gb RAM, 80GB root hard drive, and 1 - 80gb hdd each at /dev/sdb, /dev/sdc. and /dev/sdd.

## Get The Bits

Get the bits however you get them, and unpack the initial archive.  You should have 4 files that come from it:
* konvoy
* konvoy_vX.Y.Z.tar
* konvoy-preflight
* konvoy-base-centos7_v1.0.0

create a "Konvoy" directory in your Applications folder andf move the files into there.  This way, as new versions come out, you can just dump the new files into the same directory.

Add that directory to your "PATH" by running the following command.  
```
export PATH=/Applications/konvoy:$PATH
```

You should probably add it to your bash profile as well to ensure that the directory is picked up every time you open a new Terminal session.

## Prepare your Installation Environment

### Set Up the Drives
This step assumes that the 3 additional drives are un-initialized, and unformatted.  It also assumes that the drives are available at /dev/sdb, /dev/sdc, and /dev/sdd.  For a successful deployment, each drive MUST be 55GB or greater in size as Prometheus tries to grab 50 GB of space.  You will first need to ssh into each worker node to accomplish the following.  I found iTerm's broadcast function to be very useful to accomplish this:

#### Create the File System on each drive (one at a time)
```
sudo mkfs.ext4 /dev/sdb
#confirm with a y

sudo mkfs.ext4 /dev/sdc
#confirm with a y

sudo mkfs.ext4 /dev/sdd
#confirm with a y
```

#### Create the /mnt/disks directory
```
sudo mkdir /mnt/disks
```

#### Mount Drives & Modify "fstab" (one at a time).  
Do this fo each drive /dev/sdb, /dev/sdc, /dev/sdd.  First you will need to ssh into the nodes with the appropriate key
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

### Prepare your Working Directory
Create a folder somewhere on your local machine and navigate to it.

Create "Inventory.yaml"
Create this file in your working directory.  It should look something like below:
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
    ansible_ssh_private_key_file: "key_file_location"
    ansible_user: "userid"
    control_plane_endpoint: ""
    order: sorted
```

#### Initialize Konvoy Environment 
(as we will be doing this on prem, we will need to "SKIP_AWS")
```
export SKIP_AWS=true
./konvoy init --provisioner=none [--cluster-name honey-badger]
```

#### Modify the cluster.yaml accordingly
The init process will create a "cluster.yaml" file in the working directory.  Modify it to fit your environment.  Specific fields you will need to modify are below.  Please see the Konvoy documentation for additional configuration options.
* controlPlaneEndpointOverride (This is the load balanced address and port that the Control Plane will be exposed over.  Make sure that it does not conflict with any addresses on your LAN.)
* podSubnet (make sure it does not overlap your existing LAN)
* metallb addresses (this is the range of addresses metallb will hand out for services.  Make sure they do not overlap existing addresses on your LAN)

#### Run Preflight
```
./konvoy check preflight
```

## Build the Kluster
```
./konvoy up
```
Let it run.  There is a good chance that it will timeout at the beginning of the addons section. with an error like this:
```
STAGE [Deploying Enabled Addons]

Error: failed to deploy the cluster: error generating workflow from Addon list: failed to create workflow for https://github.com/mesosphere/kubeaddons-configs@v0.0.29: failed to deploy addon CRD opsportal; error: deployConfig failed: exit status 1 [stderr: error: unable to recognize "STDIN": no matches for kind "Addon" in version "kubeaddons.mesosphere.io/v1alpha1"
]
exit status 1
```
If so, run `./konvoy up` again and it will cycle through the already completed steps and proceed to the addon deployment

## Set KubeConfig File
use the full path so that if you change directories, it will still work.
```
export KUBECONFIG=/Users/dcmennell/clusters/konvoy/badger/konvoy_v0.4.0/admin.conf
```

## Open Ops Portal in Browser
thanks James D for putting together this cool command to automatically open the Konvoy Ops Portal (run it in the CLI)
```
open https://$(./konvoy get ops-portal | grep Username | awk '{print $2}'):$(./konvoy get ops-portal | grep Password | awk '{print $2}')@$(./konvoy get ops-portal | grep https | cut -f3 -d\/)
```

## Expose Kommander
```
kubectl expose deployment kommander-deployment -n=kommander --type=LoadBalancer --name=k7r-dash
```
Find the address the service was exposed over:
```
kubectl get services -n=kommander
```

