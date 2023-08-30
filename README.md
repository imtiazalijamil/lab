# Multi node k8s cluster with LoadBalancer (Metallb)

- Install kubectl
- Install KinD
- Create kinD config file for multinode setup
- Create multinode KinD cluster
- Install metallb via manifest file
- Define metallb address pool and L2advertisement
- deploy a test service to test the LoadBalancer

#### Install kubectl on local machine to intract with kubernetes cluster
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### Install KinD
- kinD Reference [KinD Documentation](https://kind.sigs.k8s.io/)
```
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

##### KinD config to create multi-node cluster

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```

#### Create KinD Cluster with config file

```shell
kind create cluster --config kind-config.yml
```

#### Install Metallb

[Metallb in KinD](https://kind.sigs.k8s.io/docs/user/loadbalancer/)

##### MetalLB manifest

```shell
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
```
- verify the readiness
```shell
kubectl wait --namespace metallb-system \
                --for=condition=ready pod \
                --selector=app=metallb \
                --timeout=90s
```

##### Setup address pool for LoadBalancer

- Download metallb sample config
```
https://kind.sigs.k8s.io/examples/loadbalancer/metallb-config.yaml
```
- verify the docker address pool
```shell
docker network inspect -f '{{.IPAM.Config}}' kind
```
- apply the update config
```shell
kubectl apply -f metallb-config.yaml
```

## create test pods with service as load balancer 

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: foo-app
  labels:
    app: http-echo
spec:
  containers:
  - name: foo-app
    image: hashicorp/http-echo:0.2.3
    args:
    - "-text=foo"
---
kind: Pod
apiVersion: v1
metadata:
  name: bar-app
  labels:
    app: http-echo
spec:
  containers:
  - name: bar-app
    image: hashicorp/http-echo:0.2.3
    args:
    - "-text=bar"
---
kind: Service
apiVersion: v1
metadata:
  name: foo-service
spec:
  type: LoadBalancer
  selector:
    app: http-echo
  ports:
  # Default port used by the image
  - port: 5678

```
kubectl apply -f https://kind.sigs.k8s.io/examples/loadbalancer/usage.yaml

## Extract the LB ip in veriable

```shell 
LB_IP=$(kubectl get svc/foo-service -o=jsonpath='{.status.loadBalancer.ingress[0].ip}') 
```

## Test the load balancer
```shell
for _ in {1..10}; do
  curl ${LB_IP}:5678
done
```
