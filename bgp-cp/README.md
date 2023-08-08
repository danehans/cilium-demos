# WIP: BGP Control Plane

This is a demonstration of using Cilium BGP Control Plane (BGP CP) to advertise
routes to an external BGP speaking router. Each cluster advertises a service
using BGP CP and LB IPAM Pools.

## Dependencies
1. kubectl
2. kind
3. helm
4. cilium-cli

## Install the Kubernetes clusters

Use kind to create the first cluster:

```sh
$ kind create cluster --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true   # do not install kindnet
  kubeProxyMode: none       # do not run kube-proxy
  podSubnet: "10.241.0.0/16"
  serviceSubnet: "10.11.0.0/16"
name: cilium
nodes:
- role: control-plane
- role: worker
- role: worker
EOF
```

Create a Docker network that will be used for the second cluster:

```sh
docker network create -d=bridge \
    -o "com.docker.network.bridge.enable_ip_masquerade=true" \
    --attachable \
    "kind2"
```

Set the `KIND_EXPERIMENTAL_DOCKER_NETWORK` env var to have kind use this network
instead of the default `kind` network:

```sh
export KIND_EXPERIMENTAL_DOCKER_NETWORK=kind2
```

Create the second cluster:

```sh
$ kind create cluster --config - <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true   # do not install kindnet
  kubeProxyMode: none       # do not run kube-proxy
  podSubnet: "10.242.0.0/16"
  serviceSubnet: "10.12.0.0/16"
name: cilium2
nodes:
- role: control-plane
- role: worker
- role: worker
EOF
```

You now have two k8s clusters on seperate networks that are unable to communicate with each other.

## Install Cilium in the clusters

Install Cilium v1.14 in the second cluster. Kind should have the kubectl context set for the
`kind-cilium2` context. Use the `kubectl config use-context` command throughout this guide to
change between `kind-cilium` and `kind-cilium2` contexts, e.g. `cilium` and `cilium2` clusters.

```sh
$ helm upgrade --kube-context kind-cilium2 --install cilium cilium/cilium --namespace kube-system --version 1.14.0 --values - <<EOF
kubeProxyReplacement: strict
k8sServiceHost: cilium2-control-plane # use master node in kind network
k8sServicePort: 6443               # use api server port
hostServices:
  enabled: false
externalIPs:
  enabled: true
nodePort:
  enabled: true
hostPort:
  enabled: true
image:
  pullPolicy: IfNotPresent
ipam:
  mode: kubernetes
tunnel: disabled
ipv4NativeRoutingCIDR: 10.12.0.0/16
bgpControlPlane:
  enabled: true
autoDirectNodeRoutes: true
EOF
```

Wait for Cilium to report a "ready" status:

```sh
$ cilium --context kind-cilium2 status --wait
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    disabled (using embedded mode)
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

DaemonSet              cilium             Desired: 2, Ready: 2/2, Available: 2/2
Deployment             cilium-operator    Desired: 2, Ready: 2/2, Available: 2/2
Containers:            cilium-operator    Running: 2
                       cilium             Running: 2
Cluster Pods:          3/3 managed by Cilium
Helm chart version:    1.14.0
Image versions         cilium             quay.io/cilium/cilium:v1.14.0@sha256:5a94b561f4651fcfd85970a50bc78b201cfbd6e2ab1a03848eab25a82832653a: 2
                       cilium-operator    quay.io/cilium/operator-generic:v1.14.0@sha256:3014d4bcb8352f0ddef90fa3b5eb1bbf179b91024813a90a0066eb4517ba93c9: 2
```

Install Cilium v1.14 in the first cluster:

```sh
$ helm upgrade --kube-context kind-cilium --install cilium cilium/cilium --namespace kube-system --version 1.14.0 --values - <<EOF
kubeProxyReplacement: strict
k8sServiceHost: cilium-control-plane # use master node in kind network
k8sServicePort: 6443               # use api server port
hostServices:
  enabled: false
externalIPs:
  enabled: true
nodePort:
  enabled: true
hostPort:
  enabled: true
image:
  pullPolicy: IfNotPresent
ipam:
  mode: kubernetes
tunnel: disabled
ipv4NativeRoutingCIDR: 10.11.0.0/16
bgpControlPlane:
  enabled: true
autoDirectNodeRoutes: true
EOF
```

Wait for Cilium to report a "ready" status:

```sh
$ cilium --context kind-cilium status --wait
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    disabled (using embedded mode)
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled

Deployment             cilium-operator    Desired: 2, Ready: 2/2, Available: 2/2
DaemonSet              cilium             Desired: 2, Ready: 2/2, Available: 2/2
Containers:            cilium             Running: 2
                       cilium-operator    Running: 2
Cluster Pods:          3/3 managed by Cilium
Helm chart version:    1.14.0
Image versions         cilium             quay.io/cilium/cilium:v1.14.0@sha256:5a94b561f4651fcfd85970a50bc78b201cfbd6e2ab1a03848eab25a82832653a: 2
                       cilium-operator    quay.io/cilium/operator-generic:v1.14.0@sha256:3014d4bcb8352f0ddef90fa3b5eb1bbf179b91024813a90a0066eb4517ba93c9: 2
```

## [TODO] Setup the external BGP Router

Provide steps for running Bird as a container that attaches to each network, peers
with the Cilium BGP routers and advertises routes between clusters.

Update the IPs in the the `bird.conf` file to match the node IP's in your clusters.

Run the Bird BGP router:

```sh
docker run --name bird-router \
  --network kind \
--mount type=bind,source=$GOPATH/src/github.com/danehans/cilium-demos/bgp-cp,target=/config \
--cap-add=NET_ADMIN \
-d bird-container
```

Attach the router to the network of the secon cluster:

```sh
docker network connect kind2 bird-router
```

## Run Sample Apps

Run NGINX in the first cluster to test network connectivity. If needed, set the kubectl:

```sh
$ kubectl --context kind-cilium apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF
```

Wait for the NGINX deployment to be ready:

```sh
$ kubectl --context kind-cilium wait --timeout=2m deployment/nginx --for=condition=Available

deployment.apps/nginx condition met
```

Get the IPs of the NGINX pods:

```sh
$ kubectl --context kind-cilium get po -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP             NODE             NOMINATED NODE   READINESS GATES
nginx-ff6774dc6-rnw9n   1/1     Running   0          16s   10.241.2.140   cilium-worker    <none>           <none>
nginx-ff6774dc6-x9ct9   1/1     Running   0          16s   10.241.1.188   cilium-worker2   <none>           <none>
```

Use cURL to test network connectivity between the pods:
```sh
$ kubectl --context kind-cilium exec po/nginx-ff6774dc6-rnw9n -- curl -s -o /dev/null -w "%{http_code}" http://10.241.1.188
200
```

Run NGINX in the second cluster to test network connectivity:

```sh
$ kubectl --context kind-cilium2 apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF
```

Wait for the NGINX deployment to be ready:

```sh
$ kubectl --context kind-cilium2 wait --timeout=2m deployment/nginx --for=condition=Available

deployment.apps/nginx condition met
```

Get the IPs of the NGINX pods:

```sh
$ kubectl --context kind-cilium2 get po -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP             NODE              NOMINATED NODE   READINESS GATES
nginx-65df995dc8-n8rwf   1/1     Running   0          14s   10.242.2.52    cilium2-worker2   <none>           <none>
nginx-65df995dc8-pfjh4   1/1     Running   0          14s   10.242.1.198   cilium2-worker    <none>           <none>
```

Use cURL to test network connectivity between the pods:

```sh
$ kubectl --context kind-cilium2 exec po/nginx-65df995dc8-n8rwf -- curl -s -o /dev/null -w "%{http_code}" http://10.242.1.198
200
```

Intracluster connectivity has now been successfully verified.

Expose the NGINX deployment for each cluster using a load balancer service. First create the IPAM
pool used to assign addresses to load balancer services:

```sh
kubectl --context kind-cilium apply -f - <<EOF
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: "test"
spec:
  cidrs:
  - cidr: "10.11.0.0/16"
EOF
```

Repeat for the second cluster:

```sh
kubectl --context kind-cilium2 apply -f - <<EOF
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: "test"
spec:
  cidrs:
  - cidr: "10.12.0.0/16"
EOF
```

```sh
kubectl --context kind-cilium expose deployment nginx --type=LoadBalancer --port=80 --labels app=nginx
kubectl --context kind-cilium2 expose deployment nginx --type=LoadBalancer --port=80 --labels app=nginx
```

The nginx service for each cluster should have an external IP assigned from the IPAM pool.
Check the external IP for the nginx service in the first cluster:

```sh
$ kubectl --context kind-cilium get svc/nginx
NAME    TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)        AGE
nginx   LoadBalancer   10.11.156.124   10.11.108.98   80:31712/TCP   9m10s
```

Repeat for the second cluster:

```sh
$ kubectl --context kind-cilium2 get svc/nginx
NAME    TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
nginx   LoadBalancer   10.12.153.124   10.12.141.231   80:30596/TCP   9m40s
```

The external IPs will be advertised by BGP in upcoming steps to provide intra-cluster connectivity
between services.

## Configure BGP

The Cilium BGP configuration is a two part process, annotate the nodes that will run BGP and configure
a peering policy to configure BGP peers, advertise routes, etc. Refer to the [official docs][docs] for
additional details regarding BGP Control Plane.


Annotate the nodes in the first cluster:

```sh
kubectl --context kind-cilium annotate node/cilium-control-plane cilium.io/bgp-virtual-router.65100="local-port=179"
kubectl --context kind-cilium annotate node/cilium-worker cilium.io/bgp-virtual-router.65100="local-port=179"
kubectl --context kind-cilium annotate node/cilium-worker2 cilium.io/bgp-virtual-router.65100="local-port=179"
```

By default Cilium will instantiate each virtual router without a listening port uless the `local-port`
annotation is set.

Annotate the nodes in the second cluster:

```sh
kubectl --context kind-cilium2 annotate node/cilium2-control-plane cilium.io/bgp-virtual-router.65200="local-port=179"
kubectl --context kind-cilium2 annotate node/cilium2-worker cilium.io/bgp-virtual-router.65200="local-port=179"
kubectl --context kind-cilium2 annotate node/cilium2-worker2 cilium.io/bgp-virtual-router.65200="local-port=179"
```

__Note:__ In typical deployments, BGP will not run on control-plane nodes since they do not run workload pods.

Since each cluster is under separate administration in this demo, a different autonomous system (AS) numbers
are used, `65100` and `65200` respectivly.

Get the node IPs of the second cluster that will be used for the BGP peering policy of the first cluster:

```sh
$ kubectl --context kind-cilium2 get nodes -o wide
NAME                    STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
cilium2-control-plane   Ready    control-plane   93m   v1.25.3   172.18.0.7    <none>        Ubuntu 22.04.1 LTS   5.10.124-linuxkit   containerd://1.6.9
cilium2-worker          Ready    <none>          93m   v1.25.3   172.18.0.6    <none>        Ubuntu 22.04.1 LTS   5.10.124-linuxkit   containerd://1.6.9
cilium2-worker2         Ready    <none>          93m   v1.25.3   172.18.0.5    <none>        Ubuntu 22.04.1 LTS   5.10.124-linuxkit   containerd://1.6.9
```

Apply a BGP peering policy to the first cluster:

```sh
kubectl --context kind-cilium apply -f - <<EOF
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPPeeringPolicy
metadata:
  name: demo
spec:
  nodeSelector:
    matchLabels:
      kubernetes.io/os: linux
  virtualRouters:
    - localASN: 65100
      neighbors:
        - peerASN: 65000
          peerAddress: 172.22.0.5/32
      serviceSelector:
        matchLabels:
          app: nginx
EOF
```

And for the second cluster:

```sh
kubectl --context kind-cilium2 apply -f - <<EOF
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPPeeringPolicy
metadata:
  name: demo
spec:
  nodeSelector:
    matchLabels:
      kubernetes.io/os: linux
  virtualRouters:
    - localASN: 65200
      neighbors:
        - peerASN: 65000
          peerAddress: 172.25.0.5/32
      serviceSelector:
        matchLabels:
          app: nginx
EOF
```

[docs]: https://docs.cilium.io/en/v1.14/network/bgp-control-plane/
