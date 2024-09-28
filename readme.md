# Talos Kubernetes Clusters as kubevirt vms with GatewayAPI

## Prerequisites

Have a file `external-dns.json` to hold the [configuration for external-dns](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/azure.md#configuration-file). #make sure subscription is registered with Microsoft.Storage, otherwise register with `az provider register --namespace Microsoft.Storage` (necessary for CDI PVCs). You need a VNC client to see the screen of the VM.


```bash
export CLUSTER_RG=infra
export CLUSTER_NAME=kv
export GH_TOKEN=""#a valid Github token for flux
export AZURE_CLIENT_SECRET="" #an azure client secret for external-dns and cert-manager
```

## Cluster preparation

```bash
az aks get-credentials --resource-group $CLUSTER_RG -n  $CLUSTER_NAME -f ~/Desktop/kubeconfigs/$CLUSTER_NAME-cluster.yaml --overwrite-existing
flux install
kubectl create secret generic -n flux-system kubespaces-infra-git-deploy --from-literal=username=ams0 --from-literal=password=${GH_TOKEN}

kubectl create ns external-dns
kubectl create secret generic azure-config-file -n external-dns --from-file=azure.json=/Users/alessandro/Desktop/kubeconfigs/external-dns.json
kubectl create ns cert-manager
kubectl create secret generic azuredns-config -n cert-manager --from-literal=client-secret=$AZURE_CLIENT_SECRET

# applies the manifests for installing cert-manager and external dns
kubectl apply -f $HOME/repos/github/kubespaces/kubespaces-infra/gitops/infrastructure/repos/ks-infra.yaml
kubectl apply -f $HOME/repos/github/kubespaces/kubespaces-infra/gitops/infrastructure/manifests/kustomization-ks-infra.yaml

# install Envoy gateway
helm upgrade -i eg oci://docker.io/envoyproxy/gateway-helm --version v1.1.2 -n envoy-gateway-system --create-namespace

# create a GatewayClass
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
EOF
```

## Install Kubevirt

```bash
#kubevirt
export RELEASE=$(curl https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/$RELEASE/kubevirt-operator.yaml
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/$RELEASE/kubevirt-cr.yaml
# wait until all KubeVirt components are up
kubectl -n kubevirt wait kv kubevirt --for condition=Available
```

One of the pods from a job was in pending state, so I added this label to the node

```bash
kubectl label node aks-nodepool1-17395826-vmss000002 node-role.kubernetes.io/master=true
```
## Install CDI

```bash
export VERSION=$(curl -s https://api.github.com/repos/kubevirt/containerized-data-importer/releases/latest | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')
kubectl create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$VERSION/cdi-operator.yaml
kubectl create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$VERSION/cdi-cr.yaml
```

## Upload the Talos ISO

Get the iso from the [Talos Image Factory](https://factory.talos.dev/) and upload it:

```bash
kubectl port-forward -n cdi service/cdi-uploadproxy 18443:443

virtctl image-upload \
  --image-path=/path-to-iso/talos_kubevirt.iso \
  --pvc-name=talos \
  --size=10Gi \
  --storage-class=azurefile-premium \
  --uploadproxy-url=https://127.0.0.1:18443 \
  --insecure \
  --wait-secs=240
```

This creates the talos PVC that contains the ISO to boot from.

## Deploy a VM

Create the single-VM Talos node

First you'll need an OS disk:

```bash
kubectl apply -f pvc.yaml
```

Then the actual VM:

```bash
kubectl apply -f vm.yaml
```

Check the status of the VM with VNC:

```bash
virtctl vnc taloscp
```

To destroy and star over:

```bash
kubectl delete vm taloscp
kubectl delete pvc taloshd

kubectl apply -f ../pvc.yaml
kubectl apply -f ../vm.yaml
```


## Deploy Talos

Generate a configuration:

```bash
export taloscp='taloscp.kubespaces.cloud'
talosctl gen config taloscp https://$taloscp:6443 --additional-sans $taloscp
```

Modify the configuration allowing for single node and using the /dev/vda disk
add
network:
      extraHostEntries:
        - ip: 127.0.0.1 # The IP of the host.
          aliases:
            - taloscp.kubespaces.cloud

```bash
# mac version
sed -i'' -e 's/sda/vda/g' controlplane.yaml
# sed -i'' -e 's/sda/vda/g' worker.yaml
sed -i '' 's/^    # \(allowSchedulingOnControlPlanes.*\)/    \1/' controlplane.yaml

# linux version
# sed -i 's/sda/vda/g' controlplane.yaml
# sed -i 's/^    # \(allowSchedulingOnControlPlanes.*\)/    \1/' controlplane.yaml
```

Apply the configuration:

```bash
talosctl apply-config -f controlplane.yaml -n $taloscp -e $taloscp  --insecure #remove insecure after the first apply
```

export TALOSCONFIG=./talosconfig
talosctl config endpoint $taloscp
talosctl config node $taloscp
talosctl bootstrap --talosconfig=./talosconfig
talosctl kubeconfig ./kubeconfig


```bash
talosctl health
talosctl services
```