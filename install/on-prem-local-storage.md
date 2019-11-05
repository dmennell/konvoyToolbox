# Konvoy Install CHEATSHEET - OnPrem with Local Storage
these instructions are what I have used to deploy Konvoy in an on prem environment with Local storage.  It has been validated with the following environment:
* OSX Mojave Install Node.
* Centos 7.6 Minimal Hyper-V based VM's.  
* 3 x Master node contains 8vcpu, 16gb RAM, and 80GB root hard drive. 
* 6 x Worker nodes have 8vcpu, 16gb RAM, 80GB root hard drive, and 3 - 80gb hdd each at /dev/sdb, /dev/sdc. and /dev/sdd.
* SSH key copied to each of the cluster nodes for automated install

## Prerequisites
Install needed software and get the files into place

### Installer Node Prerequisites
Install the following software packages from your favorite package repo:
* Docker Desktop (18.09.2 or newer) - https://www.docker.com/products/docker-desktop
* Kubernetes CLI (15.3 or newer) - https://kubernetes.io/docs/tasks/tools/install-kubectl/

### Cluster Node (Master & Kubelet) Prerequisites and Preparation
Cluster Node preparation is really very simple.  Beyond a "Minimal" version of the install, there are no additional packages needed.  Konvoy takes care of installing all required prerequisite software.

Run the following on all cluster nodes (Masters and Kubelets).  You will first need to ssh into each worker node to accomplish the following.  I found iTerm's broadcast function to be very useful to accomplish this.

DO NOT make any additional package or configuration changes.  The Konvoy installer will make all necessary changes and addithions via the Ansible installer.

#### Turn-Off Passwords for SUDO Commands.
This is a convenience step that is not a requirement.  It just makes the installation easier.
```
sudo visudo
```
remove the comment "#" next to the line that specifies what members of "wheel should not be prompted for password then save the file. "esc", ":", "wq"

#### Disable SWAP and Firewalld
Prior to installing, you will need to make sure that both firewalld and SWAP are turned of & disabled:
```
sudo systemctl stop firewalld && sudo systemctl disable firewalld
sudo swapoff -a
```
you will also need to remove any row entries that contain the word "swap" from the "/etc/fstab" directory.
```
sudo vi /etc/fstab
```
navigate cursor to any row that contains swap and "dd" do delete that row.  Now "esc", ":", "wq" to save and exit

#### Reboot cluster node
```
sudo reboot
```

### Drive Setup on Kubelet Nodes
Run the following on all Kubelet nodes in your cluster.

This step assumes that the 3 additional drives are un-initialized, and unformatted.  It also assumes that the drives are available at /dev/sdb, /dev/sdc, and /dev/sdd.  For a successful deployment, each drive MUST be 55GB or greater in size as Prometheus tries to grab 50 GB of space.  You will first need to ssh into each worker node to accomplish the following.  I found iTerm's broadcast function to be very useful to accomplish this:

#### Create the File System on each drive (for CentOS)
```
sudo mkfs.ext4 -F /dev/sdb
sudo mkfs.ext4 -F /dev/sdc
sudo mkfs.ext4 -F /dev/sdd
```

#### Create the File System on each drive (for Ubuntu - the default device names are different)
```
sudo mkfs.ext4 -F /dev/xvdb
sudo mkfs.ext4 -F /dev/xvdc
sudo mkfs.ext4 -F /dev/xvdd
```

#### Create the /mnt/disks directory
```
sudo mkdir /mnt/disks
```

#### Mount Drives & Modify "fstab" (one at a time - CentOS Version).  
Do this fo each drive /dev/sdb, /dev/sdc, /dev/sdd.  First you will need to ssh into the nodes with the appropriate key
```
#Mount /dev/sdb
DISK_UUID=$(sudo blkid -s UUID -o value /dev/sdb)
sudo mkdir /mnt/disks/$DISK_UUID
sudo mount -t ext4 /dev/sdb /mnt/disks/$DISK_UUID
echo UUID=`sudo blkid -s UUID -o value /dev/sdb` /mnt/disks/$DISK_UUID ext4 defaults 0 2 | sudo tee -a /etc/fstab
sleep 2
#Mount /dev/sdc
DISK_UUID=$(sudo blkid -s UUID -o value /dev/sdc)
sudo mkdir /mnt/disks/$DISK_UUID
sudo mount -t ext4 /dev/sdc /mnt/disks/$DISK_UUID
echo UUID=`sudo blkid -s UUID -o value /dev/sdc` /mnt/disks/$DISK_UUID ext4 defaults 0 2 | sudo tee -a /etc/fstab
sleep 2
#Mount /dev/sdd
DISK_UUID=$(sudo blkid -s UUID -o value /dev/sdd)
sudo mkdir /mnt/disks/$DISK_UUID
sudo mount -t ext4 /dev/sdd /mnt/disks/$DISK_UUID
echo UUID=`sudo blkid -s UUID -o value /dev/sdd` /mnt/disks/$DISK_UUID ext4 defaults 0 2 | sudo tee -a /etc/fstab
```

#### Mount Drives & Modify "fstab" (one at a time - Ubuntu Version).  
Do this fo each drive /dev/sdb, /dev/sdc, /dev/sdd.  First you will need to ssh into the nodes with the appropriate key
```
#Mount /dev/xvdb
DISK_UUID=$(sudo blkid -s UUID -o value /dev/xvdb)
sudo mkdir /mnt/disks/$DISK_UUID
sudo mount -t ext4 /dev/xvdb /mnt/disks/$DISK_UUID
echo UUID=`sudo blkid -s UUID -o value /dev/xvdb` /mnt/disks/$DISK_UUID ext4 defaults 0 2 | sudo tee -a /etc/fstab
sleep 2
#Mount /dev/xvdc
DISK_UUID=$(sudo blkid -s UUID -o value /dev/xvdc)
sudo mkdir /mnt/disks/$DISK_UUID
sudo mount -t ext4 /dev/xvdc /mnt/disks/$DISK_UUID
echo UUID=`sudo blkid -s UUID -o value /dev/xvdc` /mnt/disks/$DISK_UUID ext4 defaults 0 2 | sudo tee -a /etc/fstab
sleep 2
#Mount /dev/xvdd
DISK_UUID=$(sudo blkid -s UUID -o value /dev/xvdd)
sudo mkdir /mnt/disks/$DISK_UUID
sudo mount -t ext4 /dev/xvdd /mnt/disks/$DISK_UUID
echo UUID=`sudo blkid -s UUID -o value /dev/xvdd` /mnt/disks/$DISK_UUID ext4 defaults 0 2 | sudo tee -a /etc/fstab
```

### Get The Bits
Get the bits however you get them, and unpack the initial archive.  You should have 4 files that come from it:
* konvoy
* konvoy_vX.Y.Z.tar
* konvoy-preflight
* konvoy-base-centos7_vX.Y.Z

create a "Konvoy" directory in your "/Applications" folder and move the files into there.  This way, as new versions come out, you can just dump the new files into the same directory without needing to change anything else.

Add that directory to your "PATH" by running the following command.  
```
export PATH=/Applications/konvoy:$PATH
```
You should probably add it to your bash profile as well to ensure that the directory is picked up every time you open a new Terminal session.

## Prepare your Working Directory
Create a folder somewhere on your local machine and navigate to it.

### Initialize Konvoy Environment 
This will create the files needed for environment provisioning.  As we will be doing this on prem, we will need to "SKIP_AWS"
```
export SKIP_AWS=true
konvoy init --provisioner=none [--cluster-name <cluster-name>]
```
This will create 2 files in particular:
* cluster.yaml
* inventory.yaml

### Modify Your "inventory.yaml" File
You will need to specify the IP addresses associated with the Control Plane, the Worker Nodes, and the SSH connection information:
* Enter the IP Addresses for your "Control Plane" Nodes
* Enter the IP Addresses for your "Worker" nodes
* Specify your "ansible_port" (SSH default is 22)
* Specify the location of your SSH "Private Key"
* specify the "ansible_user" user ID (SSH Login Information)

### Modify the "cluster.yaml"
Specific fields you will need to modify are below.  Please see the Konvoy documentation for additional configuration options.
* controlPlaneEndpointOverride:
  * This is the load balanced address and port that the Control Plane will be exposed over (ipAddress:6443)
  * Make sure that it does not conflict with any addresses on your LAN.
  * podSubnet (make sure it does not overlap your existing LAN)
* metallb addresses:
  * this is the range of LAN addresses metallb will hand out for services (minIpAddress-maxIpAddress) 
  * Make sure they are not handed out by your DHCP Server, and that they are not already in use.

## Build the Kluster
Now we are ready to run the preflight checks and build the cluster.

### Run Preflight
```
konvoy check preflight
```
Assuming everything is correct, the preflight check will exit normally.

### Build the Cluster
```
konvoy up -y
```
The "-y" flag bypasses the confirmation that this will take about 15 minutes.Let it run.  Based on many contributing factors (bandwith, latency, packet-loss) between installer & servers, There is a chance that it will timeout at the beginning of the addons sectionwith an error like this:
```
STAGE [Deploying Enabled Addons]

Error: failed to deploy the cluster: error generating workflow from Addon list: failed to create workflow for https://github.com/mesosphere/kubeaddons-configs@v0.0.29: failed to deploy addon CRD opsportal; error: deployConfig failed: exit status 1 [stderr: error: unable to recognize "STDIN": no matches for kind "Addon" in version "kubeaddons.mesosphere.io/v1alpha1"
]
exit status 1
```
If so, run `konvoy up` again and it will cycle through the already completed steps and proceed to the addon deployment.

Assuming the install completes correctly, you will see a message that provides the URL for "Ops Landing Page" and the UserID and password needed to access it.
```
"https://ip-address/ops/landing"
```

## Set KubeConfig File
use the full path so that if you change directories, it will still work.
```
konvoy apply kubeconfig
```

### Verify "kubectl" access to cluster
Verify that you can get to the API server via kubectl
```
kubectl get nodes
```

If you lose your credentials, run the following command from your install node (make sure that you are in your install directory.
```
konvoy get ops-portal
```


YOU ARE DONE... HAVE FUN WITH KUBERNETES!!!
