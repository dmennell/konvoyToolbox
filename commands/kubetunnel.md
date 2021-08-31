
PreWork

You will need kubectl contexts for both the kommander and cluster 2b managed available
(kubectl config get-contexts)

Create the Workspace and Namespace in the Kommander UI.  Do not create the Namespace through kubectl, and set the local namespace variable

Create directories for each cluster to be managed via Kubetunnel, and use these as the working directories whenever creating your gateways and tunnels


==================================================
Use kubectl context for KOMMANDER CLUSTER 
(kubectl config use-context <kommander-cluster-context>)
==================================================

Obtain the hostname and CA certificate for the Kommander cluster:
```
hostname=$(kubectl get service -n kubeaddons traefik-kubeaddons -o go-template='{{with index .status.loadBalancer.ingress 0}}{{or .hostname .ip}}{{end}}')
b64ca_cert=$(kubectl get secret -n cert-manager kubernetes-root-ca -o=go-template='{{index .data "tls.crt"}}')
```

Set the namespace environment variable
(use the namespace that corresponds to the created/chosen workspace)
```
namespace=<kommander-workspace-namespace>
```

...and verify it is the same as the Kommander workspace namespace
```
kubectl get namespace ${namespace}
```

### CREATE A TUNNEL GATEWAY

Create environment variables for ca_cedrt secret and GTunnel Gateway...
```
cacert_secret=kubetunnel-ca
gateway=<gateway-name>
```

...Create Tunnel Gateway file...
```
cat > gateway.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  namespace: ${namespace}
  name: ${cacert_secret}
data:
  ca.crt:
    ${b64ca_cert}
---
apiVersion: kubetunnel.d2iq.io/v1alpha1
kind: TunnelGateway
metadata:
  namespace: ${namespace}
  name: ${gateway}
spec:
  ingress:
    caSecretRef:
      namespace: ${namespace}
      name: ${cacert_secret}
    loadBalancer:
      hostname: ${hostname}
    urlPathPrefix: /ops/portal/kubetunnel
    extraAnnotations:
      kubernetes.io/ingress.class: traefik
      traefik.frontend.rule.type: PathPrefixStrip
EOF
```

...apply to the Kommander cluster...
```
kubectl apply -f gateway.yaml
```

...and verify that it exists
kubectl get tunnelgateway -n ${namespace} ${gateway}


### CREATE TUNNEL CONNECTOR

Set tunnel connector environment variable
```
connector=<unique-tunnel-connector-name>
```

Create connector YAML file
```
cat > connector.yaml <<EOF
apiVersion: kubetunnel.d2iq.io/v1alpha1
kind: TunnelConnector
metadata:
  namespace: ${namespace}
  name: ${connector}
spec:
  gatewayRef:
    name: ${gateway}
EOF
```

Apply file to Kommander cluster
```
kubectl apply -f connector.yaml
```

and verify that it exists and is "LISTENING"
```
kubectl get tunnelconnector -n ${namespace} ${connector}
```

Once it is "LISTENING", generate the "manifest.yaml" file
```
while [ "$(kubectl get tunnelconnector -n ${namespace} ${connector} -o jsonpath="{.status.state}")" != "Listening" ]
do
  sleep 5
done
manifest=$(kubectl get tunnelconnector -n ${namespace} ${connector} -o jsonpath="{.status.tunnelAgent.manifestsRef.name}")
while [ -z ${manifest} ]
do
  sleep 5
  manifest=$(kubectl get tunnelconnector -n ${namespace} ${connector} -o jsonpath="{.status.tunnelAgent.manifestsRef.name}")
done
kubectl get secret -n ${namespace} ${manifest} -o jsonpath='{.data.manifests\.yaml}' | base64 -d > manifest.yaml
```

==================================================
Use kubectl context for CLUSTER 2B MANAGED
==================================================

apply the manifest file
```
kubectl apply -f manifest.yaml
```

Watch kubetunnel pods and wait for job to complete
```
watch kubectl get pods -n kubetunnel
```

==================================================
Use kubectl context for KOMMANDER CLUSTER
==================================================


Wait for the tunnel to be connected by the tunnel agent
```
while [ "$(kubectl get tunnelconnector -n ${namespace} ${connector} -o jsonpath="{.status.state}")" != "Connected" ]
do
  sleep 5
done
```

* Add the cluster to Kommander 

create environment variables...
```
managed=<cluster-display-name>
display_name=${managed}
```

...create the kommander YAML File...
```
cat > kommander.yaml <<EOF
apiVersion: kommander.mesosphere.io/v1beta1
kind: KommanderCluster
metadata:
  namespace: ${namespace}
  name: ${managed}
  annotations:
    kommander.mesosphere.io/display-name: ${display_name}
spec:
  clusterTunnelConnectorRef:
    name: ${connector}
EOF
```

...apply the file to the kommander cluster...
```
kubectl apply -f kommander.yaml
```

...and wait for the managed cluster to join the Kommander cluster
```
while [ "$(kubectl get kommandercluster -n ${namespace} ${managed} -o jsonpath='{.status.phase}')" != "Joined" ]
do
  sleep 5
done
kubefed=$(kubectl get kommandercluster -n ${namespace} ${managed} -o jsonpath="{.status.kubefedclusterRef.name}")
while [ -z "${kubefed}" ]
do
  sleep 5
  kubefed=$(kubectl get kommandercluster -n ${namespace} ${managed} -o jsonpath="{.status.kubefedclusterRef.name}")
done
kubectl wait --for=condition=ready --timeout=60s kubefedcluster -n kommander ${kubefed}
kubectl get kubefedcluster -n kommander ${kubefed}
```



















