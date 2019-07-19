# Deploy Konvoy Kubernetes with Open-EBS Storage Backend

The following instructions walk you through installing Konvoy Kubernetes Cluster with OpenEBS and MayaData Storage Manager Integration

## Step 1. Launch a Konvoy cluster that uses OpenEBS.

### a. Extract the konvoy binaries.
```
tar -xvf ~/Downloads/konvoy_v0.4.0.tar.bz2
cd konvoy_v0.4.0
```

### b. Generate a cluster.yaml file.
```
eval $(maws login XXX_Mesosphere-PowerUser)
./konvoy init --provisioner=aws --cluster-name="XXX-Konvoy-OpenEBS"
```

### c. Modify your cluster.yaml to use the OpenEBS custom AMI (us-west-2, and us-east-1 only).  

* us-west-2:  ami-0f551c319ca0f39a8
* us-east-1:  ami-0cb81d9d6b65b8c52
```
vi cluster.yaml
```

Change/add the following:
```
  nodePools:
  - name: worker
    count: XX
    machine:
      imageId: ami-0f551c319ca0f39a8

  - name: control-plane
    controlPlane: true
    count: 3
    machine:
      imageId: ami-0f551c319ca0f39a8

    tags:
      owner: <CHANGE ME>
      expiration: 4h
```

### d. Launch Konvoy cluster using the Konvoy up command.
```
eval $(maws login XXX_Mesosphere-PowerUser)
./konvoy up -y
```

## Step 2. Define some OpenEBS persistent volumes on the worker nodes in your Konvoy cluster.

### a. Run the AWS CLI commands to attach some EBS volumes to the worker nodes.
```
eval $(maws login XXX_Mesosphere-PowerUser)

export CLUSTER=gpalmer_konvoy_v0.3.0-XXX     # name of your cluster, check in EC2 console
export REGION=us-west-2
export KEY_FILE=konvoy_v0.3.0-ssh.pem  


IPS=$(aws --region=$REGION ec2 describe-instances |  jq --raw-output ".Reservations[].Instances[] | select((.Tags | length) > 0) | select(.Tags[].Value | test(\"$CLUSTER-worker\")) | select(.State.Name | test(\"running\")) | [.PublicIpAddress] | join(\" \")")

export DISK_SIZE=150

aws --region=$REGION ec2 describe-instances |  jq --raw-output ".Reservations[].Instances[] | select((.Tags | length) > 0) | select(.Tags[].Value | test(\"$CLUSTER-worker\")) | select(.State.Name | test(\"running\")) | [.InstanceId, .Placement.AvailabilityZone] | \"\(.[0]) \(.[1])\"" | while read instance zone; do
echo $instance $zone
volume=$(aws --region=$REGION ec2 create-volume --size=$DISK_SIZE --volume-type gp2 --availability-zone=$zone --tag-specifications="ResourceType=volume,Tags=[{Key=string,Value=$CLUSTER}, {Key=owner,Value=michaelbeisiegel}]" | jq --raw-output .VolumeId)
sleep 10
aws --region=$REGION ec2 attach-volume --device=/dev/xvdc --instance-id=$instance --volume-id=$volume
done
```

### b. Install the OpenEBS K8s operators
```
curl -O https://openebs.github.io/charts/openebs-operator-1.0.0.yaml
```

vi openebs-operator-1.0.0.yaml


       Change line to: 

            exclude: "loop,/dev/fd0,/dev/sr0,/dev/ram,/dev/dm-,/dev/md,/dev/disk1s1"

       where "/dev/disk1s1" is the device file for the root filesystem that you see when you SSH into one of your Konvoy worker nodes and run the "df -h" command (or look in the /etc/fstab file).

     kubectl apply -f openebs-operator-1.0.0.yaml

### c. Define an OpenEBS storage class
```
cat <<EOF | kubectl apply -f -
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: openebs-jiva-default
  selfLink: "/apis/storage.k8s.io/v1/storageclasses/openebs-jiva-default"
  annotations:
    cas.openebs.io/config: |
      - name: ReplicaCount
        value: "3"
      - name: StoragePool
        value: default
      #- name: TargetResourceLimits
      #  value: |-
      #      memory: 1Gi
      #      cpu: 100m
      #- name: AuxResourceLimits
      #  value: |-
      #      memory: 0.5Gi
      #      cpu: 50m
      #- name: ReplicaResourceLimits
      #  value: |-
      #      memory: 2Gi
    openebs.io/cas-type: jiva
    storageclass.kubernetes.io/is-default-class: 'true'
provisioner: openebs.io/provisioner-iscsi
reclaimPolicy: Delete
volumeBindingMode: Immediate
EOF
```


## Step 3. Modify your Kubernetes YAML file to use the new storage class "openebs-jiva-default". 
Here is an example for Jenkins.  Launch your workload with the appropriate kubectl commands.
```
$ cat jenkins.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: jenkins-claim
  namespace: jenkins
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: openebs-jiva-default
  resources:
    requests:
      storage: 10Gi
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: jenkins-app
    spec:
      securityContext:
        fsGroup: 1000
      containers:
        - name: jenkins
          imagePullPolicy: IfNotPresent
          image: jenkins/jenkins:lts
          env:
            - name: JENKINS_URL
              value: https://<CHANGE ME>/jenkins
            - name: JENKINS_OPTS
              value: --prefix=/jenkins
          ports:
            - containerPort: 8080
            - containerPort: 50000
          volumeMounts:
            - mountPath: /var/jenkins_home
              name: jenkins-home
      volumes:
        - name: jenkins-home
          persistentVolumeClaim:
            claimName: jenkins-claim
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins-svc
  namespace: jenkins
spec:
  ports:
    - name: http
      port: 8080
    - name: agent
      port: 50000
  selector:
    app: jenkins-app
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: jenkins-ingress
  namespace: jenkins
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.frontend.rule.type: PathPrefix
spec:
  rules:
  - http:
      paths:
      - path: /jenkins
        backend:
          serviceName: jenkins-svc
          servicePort: 8080
```

## Step 4. 

TODO: Create mulitple OpenEBS storage classes that have different setups, including different replica settings and use it to test a workload failing on one worker node and re-launching on another worker node, but still attaching to an OpenEBS volume that was replicated on other nodes.

Step 5. Register your OpenEBS enabled cluster with MayaData's free tier Storage Management product at https://mayaonline.io

     a. Point your browser to https://mayaonline.io

     b. Login using Git or Google

     c. In the list of K8s cluster types, click on "Others" (they don't yet have Konvoy listed).

     d. You will be presented with a customer "kubectl apply -f" command. Copy and paste it to your kubectl session and run it.

     e. Go back to the mayaonline.io website click on the "Clusters" link.

     f. Click on the "Topologies" link and view the volumes and any snapshots or replicas that are defined as well as storage activity and health.

