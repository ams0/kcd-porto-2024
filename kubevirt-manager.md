# Deploy kubevirt-manager

```bash
git clone https://github.com/kubevirt-manager/kubevirt-manager
cd kubevirt-manager
kubectl apply -f kubernetes/ns.yaml
kubectl apply -f kubernetes/crd.yaml
kubectl apply -f kubernetes/rbac.yaml
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/pc.yaml
kubectl apply -f kubernetes/service.yaml
```