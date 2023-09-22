# BGP Control Plane Route Policies

This is a demonstration of using Cilium BGP Control Plane (BGP CP) route policies to
set path attributes of route advertisements.

## Dependencies
1. [kubectl](https://kubernetes.io/docs/tasks/tools/)
2. [kind](https://kind.sigs.k8s.io/)
3. [helm](https://helm.sh/docs/intro/install/)
4. [cilium-cli](https://github.com/cilium/cilium-cli)
5. `git clone https://github.com/cilium/cilium.git && cd cilium`

## Create a Kubernetes Cluster

Use kind to create the cluster:

```sh
make kind
```

At this time, a Cilium release has not been cut that includes route policies,
so build the cilium agent/operator images and install them in the cluster:

```sh
make kind-image
```

## Install Cilium

Create an install values file, `vals.yaml`, with BGP, local image pull, etc. configured:

```sh
ipam:
  mode: kubernetes
bgpControlPlane:
  enabled: true
tunnel: disabled
ipv4NativeRoutingCIDR: 10.0.0.0/8
autoDirectNodeRoutes: true
debug:
  enabled: true
image:
  repository: localhost:5000/cilium/cilium-dev
  useDigest: false
  tag: local
  pullPolicy: Never
operator:
  image:
    repository: localhost:5000/cilium/operator
    useDigest: false
    suffix: ""
    tag: local
    pullPolicy: Never
```

Install Cilium using the values file:

```sh
helm install cilium -n kube-system install/kubernetes/cilium/ -f vals.yaml
```

Wait for Cilium to report a "ready" status:

```sh
$ cilium status --wait
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

## Configure BGP CP

Annotate the nodes for BGP CP, creating iBGP peers between `kind-control-plane` and `kind-worer` nodes:

```sh
kubectl annotate node/kind-control-plane cilium.io/bgp-virtual-router.65100="local-port=179"
kubectl annotate node/kind-worker cilium.io/bgp-virtual-router.65100="local-port=179"
```

__Note:__ iBGP is being used since the `localPreference` attribute is not applicable for eBGP peering.

Get the node internal IPs used for BGP peering:

```sh
$ kubectl get nodes -o wide
NAME                 STATUS   ROLES           AGE    VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION      CONTAINER-RUNTIME
kind-control-plane   Ready    control-plane   132m   v1.27.1   172.25.0.3    <none>        Debian GNU/Linux 11 (bullseye)   5.15.0-78-generic   containerd://1.6.21
kind-worker          Ready    <none>          132m   v1.27.1   172.25.0.2    <none>        Debian GNU/Linux 11 (bullseye)   5.15.0-78-generic   containerd://1.6.21
```

Set the node IP env vars:

```sh
CP_IP=172.25.0.2
WORKER_IP=172.25.0.3
```

Apply the BGP peering policies:

```sh
kubectl apply -f - <<EOF
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPPeeringPolicy
metadata:
  name: cp
spec:
  nodeSelector:
    matchLabels:
      kubernetes.io/hostname: kind-control-plane
  virtualRouters:
    - exportPodCIDR: true
      serviceSelector:
        matchLabels:
          app: nginx
      localASN: 65100
      neighbors:
      - peerASN: 65100
        peerAddress: $WORKER_IP/32
        advertisedPathAttributes:
        - selectorType: CiliumLoadBalancerIPPool
          selector:
            matchLabels:
              foo: bar
          communities:
            standard:
            - 65001:100
        - selectorType: PodCIDR
          localPreference: 150
---
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPPeeringPolicy
metadata:
  name: worker
spec:
  nodeSelector:
    matchLabels:
      kubernetes.io/hostname: kind-worker
  virtualRouters:
    - exportPodCIDR: false
      localASN: 65100
      neighbors:
      - peerASN: 65100
        peerAddress: $CP_IP/32
EOF
```

The CiliumBGPPeeringPolicy `cp` is applied to the `kind-control-plane` node. This policy adds
the `communities` attribute to all CiliumLoadBalancerIPPools routes matching label `foo: bar`
and adds a local preference of `150` to Pod CIDR route.

Check the BGP peering status:

```sh
$ cilium bgp peers
Node                 Local AS   Peer AS   Peer Address   Session State   Uptime   Family         Received   Advertised
kind-control-plane   65100      65200     172.25.0.2     established     51s      ipv4/unicast   0          1
                                                                                  ipv6/unicast   0          0
kind-worker          65200      65100     172.25.0.3     established     51s      ipv4/unicast   1          0
                                                                                  ipv6/unicast   0          0
```

The session state should be `established`. Verify the details of the advertised route.

```sh
$ k get po -n kube-system -o wide | grep cil
cilium-459m5   1/1     Running   0          14m   172.25.0.3    kind-worker          <none>           <none>
cilium-9h9xw   1/1     Running   0          14m   172.25.0.2    kind-control-plane   <none>           <none>
...

$ kubectl exec -it po/cilium-9h9xw -n kube-system -c cilium-agent -- cilium bgp routes advertised ipv4 unicast peer 172.25.0.3
VRouter   Prefix          NextHop      Age     Attrs
65100     10.244.0.0/24   172.25.0.2   3m53s   [{Origin: i} {AsPath: } {Nexthop: 172.25.0.2} {LocalPref: 150}]
```

Only the pod CIDR route is being advertised by `kind-control-plane` node and has a local preference of `150`
set by CiliumBGPPeeringPolicy `cp`.

## Run a Sample App

Run nginx to test the `communities` route policy configured in CiliumBGPPeeringPolicy `cp`:

```sh
$ kubectl apply -f - <<EOF
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

Wait for the nginx deployment to be ready:

```sh
kubectl wait --timeout=2m deployment/nginx --for=condition=Available
```

Get the IPs of the nginx pods:

```sh
$ kubectl get po -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP             NODE                 NOMINATED NODE   READINESS GATES
nginx-55f598f8d-4h84s   1/1     Running   0          28s   10.244.1.245   kind-worker          <none>           <none>
nginx-55f598f8d-nkl8c   1/1     Running   0          28s   10.244.0.182   kind-control-plane   <none>           <none>
```

Use cURL to test network connectivity between the pods:
```sh
$ kubectl exec po/nginx-55f598f8d-4h84s -- curl -s -o /dev/null -w "%{http_code}" http://10.244.0.182
200
```

Connectivity within the cluster has been successfully verified and you can now
proceed with the demo.

Expose the nginx deployment using a load balancer service. First, create the load balancer
IP pool used for assigning addresses to load balancer services:

```sh
kubectl apply -f - <<EOF
apiVersion: "cilium.io/v2alpha1"
kind: CiliumLoadBalancerIPPool
metadata:
  name: demo
  labels:
    foo: bar
spec:
  cidrs:
  - cidr: "192.168.100.0/24"
EOF
```

__Note:__ The labels match the CiliumLoadBalancerIPPool `selector` of CiliumBGPPeeringPolicy `cp`.

Expose the nginx app using a service:

```sh
kubectl expose deployment nginx --type=LoadBalancer --port=80 --labels app=nginx
```

Verify the service has been assigned an IP from CiliumLoadBalancerIPPool `demo`:

```sh
$ kubectl get svc/nginx --show-labels
NAME    TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE     LABELS
nginx   LoadBalancer   10.96.156.219   192.168.100.32   80:32168/TCP   4m28s   app=nginx
```

Verify the `kind-control-plane` node is now advertising 2 routes:

```sh
$ cilium bgp peers
Node                 Local AS   Peer AS   Peer Address   Session State   Uptime   Family         Received   Advertised
kind-control-plane   65100      65100     172.25.0.3     established     15m37s   ipv4/unicast   0          2
                                                                                  ipv6/unicast   0          0
kind-worker          65100      65100     172.25.0.2     established     15m37s   ipv4/unicast   2          0
                                                                                  ipv6/unicast   0          0
```

Exec into the `kind-control-plane` Cilium Agent to verify the details of the advertised route:

```sh
$ kubectl get po -n kube-system -o wide | grep cil
cilium-459m5   1/1     Running   0          30m   172.25.0.3    kind-worker          <none>           <none>
cilium-9h9xw   1/1     Running   0          30m   172.25.0.2    kind-control-plane   <none>           <none>
...

$ $ kubectl exec -it po/cilium-9h9xw -n kube-system -c cilium-agent -- cilium bgp routes advertised ipv4 unicast peer 172.25.0.3
VRouter   Prefix              NextHop      Age      Attrs
65100     10.244.0.0/24       172.25.0.2   20m46s   [{Origin: i} {AsPath: } {Nexthop: 172.25.0.2} {LocalPref: 150}]
65100     192.168.100.32/32   172.25.0.2   2m54s    [{Origin: i} {AsPath: } {Nexthop: 172.25.0.2} {LocalPref: 100} {Communities: 65001:100}]
```

The pod CIDR route continues to have a local preference of `150`. The nginx service route is advertised
with a `ccommunities` attriubte of `65001:100` and the default local preference of `100`.

Congratulations, you've completed the demo!
