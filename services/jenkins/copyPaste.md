#To Run The Jenkins Cluster
```
kubectl create ns jenkins
kubectl create -f jenkins.yaml
kubectl create rolebinding default-admin --clusterrole=admin --serviceaccount=jenkins:default --namespace=jenkins
```