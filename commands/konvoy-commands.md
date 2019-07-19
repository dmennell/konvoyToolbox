Expose Kommander via Load Balancer
```
kubectl expose deployment kommander-deployment -n=kommander --type=LoadBalancer --name=k7r-dash
```
