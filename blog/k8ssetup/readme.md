![Kubernetes](k8slogo.png)
# Kubernetes cluster lab with ubuntu 20.04

### Cloud-init-config
If you are using this cloud-init user data file on ubuntu 20.04 it will setup all prerequisite including. apt repo, kernel parameter, kubeadm. Just change the desired kubernetes version.
<script src="https://gist.github.com/sharmavijay86/cf86ca128a166ddd456bf0be1b95e2a6.js"></script>
**Step to follow on all nodes**

```
sudo apt-get update
```

```
sudo apt-get install containerd -y
```

```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
```

```
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
```

```
sudo apt install kubelet=1.24.7-00 kubeadm=1.24.7-00 kubectl=1.24.7-00 -y
```

```
sudo modprobe overlay
sudo modprobe br_netfilter
```
```
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```
```
sudo sysctl --system
```

**step on master node**

``` sudo kubeadm init --pod-network-cidr=10.244.0.0/16```

``` mkdir -p $HOME/.kube```

``` sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config```

``` sudo chown $(id -u):$(id -g) $HOME/.kube/config```

**join worker node ( step on worker node )**

``` kubeadm join 192.168.122.220:6443 --token gefqt9.oj3kcgubehofxbz8 ```
     ```--discovery-token-ca-cert-hash sha256:a79789ade9c95182522f55b1ab17e93cd6eac9c7eaf8b7b67a6c125bbb5f50ce ```

**deploy a pod network plugin ( on master node )**

``` 
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
```
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```
 
 ### Setup of metal LB (Optional)
Apply deployment manifests-
```
kubectl get configmap kube-proxy -n kube-system -o yaml | sed -e "s/strictARP: false/strictARP: true/" | kubectl apply -f - -n kube-system
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.6/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.6/manifests/metallb.yaml
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```
Create yaml for ip pool 
```
vim ip-pool.yaml
```
Apply the ip pool for LB. Create and modify values based on your network.
```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.122.20-192.168.122.30
```
Apply
```
kubectl apply -f ip-pool.yaml
```
## Install/enable helm binary 
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
sudo bash get_helm.sh
```
## Setup ingres as nginx
 - Daemonset
 ``` 
 helm install ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx --create-namespace=true \
    --set controller.kind=DaemonSet,controller.service.enabled=false \
    --set controller.hostNetwork=true,controller.publishService.enabled=false --namespace=ingress 
 ```
 - Deployment
 ```
 helm install ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx --namespace=ingress --create-namespace=true 
 ```

## NFS dynamic provisioner setup ( Helm Chart )
```
helm repo add nfs-subdir-external-provisioner  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner   

helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --namespace=kube-system \
    --set nfs.server=192.168.1.11 \
    --set nfs.path=/k8snfs/nfs
```
- If you wish to set the storage class as default as well Then upgrade the chart
```
helm upgrade nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner     \
     --namespace=kube-system    \
     --set nfs.server=192.168.1.11  \
     --set nfs.path=/k8snfs/nfs  \
     --set storageClass.defaultClass=true
 ```
## Setup Cert-manager
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.10.0/cert-manager.yaml
```

## Setup metrics-server  
```
kubectl apply -f https://raw.githubusercontent.com/sharmavijay86/sharmavijay86.github.io/master/blog/k8ssetup/components.yaml
```
## Setup the EFK (elastic search fluentbit & kibana) stack with helm chart

```
kubectl create ns logging
helm upgrade --install fluent-bit fluent-bit --repo=https://fluent.github.io/helm-charts
helm upgrade --install elasticsearch elasticsearch --set=replicas=3,minimumMasterNodes=1,resources.requests.cpu=100m,resources.requests.memory=1Gi,volumeClaimTemplate.resources.requets.storage=5Gi, --repo=https://helm.elastic.co -n logging

helm upgrade --install kibana kibana --set=resources.requests.cpu=100m,resources.requests.memory=500Mi,ingress.enabled=true,ingress.annotations."cert-manager\.io\/cluster-issuer"=letsencrypt-staging,ingress.hosts[0].host=kibana.k8s.mevijay.dev,ingress.hosts[0].paths[0].path=/,ingress.tls[0].secretName=kibana-tls,ingress.tls[0].hosts[0]=kibana.k8s.mevijay.dev --repo=https://helm.elastic.co -n logging
```

## Setup monitoring with prometheus and grafana

- Download the hlem chart values.yaml file for both grafana and prometheus.   
```
wget https://raw.githubusercontent.com/sharmavijay86/sharmavijay86.github.io/master/blog/k8ssetup/grafana-values.yaml
wget https://raw.githubusercontent.com/sharmavijay86/sharmavijay86.github.io/master/blog/k8ssetup/prometheus-values.yaml
```
- Updates values based on your case. mainly the ingress part and storage part.
- Run the helm commands to deploy it all.
```
helm install prometheus prometheus --repo=https://prometheus-community.github.io/helm-charts -n prometheus --create-namespace
helm install grafana grafana --repo=https://grafana.github.io/helm-charts  -f grafana-values.yaml -n prometheus
```
