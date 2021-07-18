# Multicluster Traffic Mirroring with Istio and Kind

<https://piotrminkowski.com/2021/07/12/multicluster-traffic-mirroring-with-istio-and-kind/>

```shell
## Create Kubernetes clusters with Kind
kind create cluster --name c1
kind create cluster --name c2
kind get clusters
kubectx | grep kind

## Delete Kubernetes clusters with Kind
kind delete cluster --name c1
kind delete cluster --name c2

## Install MetalLB on Kubernetes clusters
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/master/manifests/namespace.yaml --context kind-c1
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)" --context kind-c1
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/master/manifests/metallb.yaml --context kind-c1
docker network inspect -f '{{.IPAM.Config}}' kind

kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/master/manifests/namespace.yaml --context kind-c2
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)" --context kind-c2
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/master/manifests/metallb.yaml --context kind-c2

kubectl apply -f k8s/metallb-c1.yaml --context kind-c1
kubectl apply -f k8s/metallb-c2.yaml --context kind-c2

## Install Istio on Kubernetes in multicluster mode
cd istio-1.10.3/tools/certs/
make -f Makefile.selfsigned.mk root-ca
make -f Makefile.selfsigned.mk kind-c1-cacerts
make -f Makefile.selfsigned.mk kind-c2-cacerts

kubectl create namespace istio-system --context kind-c1
kubectl create secret generic cacerts -n istio-system \
      --from-file=kind-c1/ca-cert.pem \
      --from-file=kind-c1/ca-key.pem \
      --from-file=kind-c1/root-cert.pem \
      --from-file=kind-c1/cert-chain.pem \
      --context kind-c1

kubectl create namespace istio-system --context kind-c2
kubectl create secret generic cacerts -n istio-system \
      --from-file=kind-c1/ca-cert.pem \
      --from-file=kind-c1/ca-key.pem \
      --from-file=kind-c1/root-cert.pem \
      --from-file=kind-c1/cert-chain.pem \
      --context kind-c2

kubectl --context kind-c1 label namespace istio-system topology.istio.io/network=network1
istioctl install -f k8s/istio-c1.yaml --context kind-c1

kubectl --context kind-c2 label namespace istio-system topology.istio.io/network=network2
istioctl install -f k8s/istio-c2.yaml --context kind-c2


## Configure multicluster connectivity
kubectl apply -f k8s/istio-cross-gateway.yaml --context kind-c1
kubectl apply -f k8s/istio-cross-gateway.yaml --context kind-c2

istioctl x create-remote-secret --context=kind-c1 --name=kind-c1 
istioctl x create-remote-secret --context=kind-c2 --name=kind-c2 

kubectl apply -f k8s/secret1.yaml --context kind-c2
kubectl apply -f k8s/secret2.yaml --context kind-c1

## Configure Mirroring with Istio
kubectl label --context kind-c1 namespace default istio-injection=enabled
kubectl label --context kind-c2 namespace default istio-injection=enabled


## Deploy application callme-service
brew install skaffold

kubectl apply -f callme-service/k8s/ --context kind-c1
kubectl get pod --context kind-c1



kubectl apply -f caller-service/k8s/ --context kind-c2
kubectl get pod --context kind-c2

siege -r 20 -c 1 http://localhost:8080/caller/service

siege -r 20 -c 1 http://localhost:8080/callme/ping
siege -r 20 -c 1 http://localhost:8080/callme/ping-with-random-error
siege -r 20 -c 1 http://localhost:8080/callme/ping-with-random-delay

```