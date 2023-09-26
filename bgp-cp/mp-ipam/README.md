# BGP Control Plane with Multi-Pool IPAM

This is a demonstration of using Cilium BGP Control Plane (BGP CP) to advertise
[Multi-Pool IPAM][mp-ipam] CIDRs between two Cilium nodes.

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

At this time, a Cilium release has not been cut that includes BGP CP [MP IPAM support][mp-ipam-support],
so build the cilium agent/operator images and install them in the cluster:

```sh
make kind-image
```

## Install Cilium

Create an install values file, `vals.yaml`, with MP IPAM, BGP, local image pull, etc. configured:

```sh
$ cat <<EOF > vals.yaml
ipam:
  mode: multi-pool
  operator:
    autoCreateCiliumPodIPPools:
      default:
        ipv4:
          cidrs:
            - 10.10.0.0/16
          maskSize: 24
      other:
        ipv4:
          cidrs:
            - 10.20.0.0/16
          maskSize: 24
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
EOF
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

Cilium is installed and created two MP IPAM pools configured from the values file.

## Configure BGP CP

Annotate the nodes for BGP CP, creating eBGP peers between `kind-control-plane` and `kind-worer` nodes:

```sh
kubectl annotate node/kind-control-plane cilium.io/bgp-virtual-router.65100="local-port=179"
kubectl annotate node/kind-worker cilium.io/bgp-virtual-router.65200="local-port=179"
```

Get the node internal IPs used for BGP peering:

```sh
kubectl get nodes -o wide
```

Set the node IP env vars:

```sh
CP_IP=172.21.0.3
WORKER_IP=172.21.0.2
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
    - exportPodCIDR: false
      localASN: 65100
      neighbors:
      - peerASN: 65200
        peerAddress: $WORKER_IP/32
      podIPPoolSelector:
        matchLabels:
          foo: bar
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
      localASN: 65200
      neighbors:
      - peerASN: 65100
        peerAddress: $CP_IP/32
EOF
```

Check the BGP peering status:

```sh
$ cilium bgp peers
Node                 Local AS   Peer AS   Peer Address   Session State   Uptime   Family         Received   Advertised
kind-control-plane   65100      65200     172.21.0.2     established     51s      ipv4/unicast   0          0
                                                                                  ipv6/unicast   0          0
kind-worker          65200      65100     172.21.0.3     established     51s      ipv4/unicast   0          0
                                                                                  ipv6/unicast   0          0
```

The session state should be `established`. Note that no routes are being advertised by `kind-control-plane`
even though it's CiliumBGPPeeringPolicy is configured with `podIPPoolSelector`. This is because no pod IP
pools match the configured `podIPPoolSelector`:

```sh
$ kubectl get ciliumpodippools/default -o yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumPodIPPool
metadata:
  creationTimestamp: "2023-09-22T18:15:51Z"
  generation: 1
  name: default
  resourceVersion: "899"
  uid: aedf6a06-a81c-4925-987d-9b159a6922ef
spec:
  ipv4:
    cidrs:
    - 10.10.0.0/16
    maskSize: 24
```

## Advertise the Default Pool

Add the `foo: bar` label to CiliumPodIPPool `default` :

```sh
kubectl label ciliumpodippool/default foo=bar
```

Verify BGP CP is now advertising the route:

```sh
$ cilium bgp peers
Node                 Local AS   Peer AS   Peer Address   Session State   Uptime   Family         Received   Advertised
kind-control-plane   65100      65200     172.21.0.2     established     7m20s    ipv4/unicast   0          1
                                                                                  ipv6/unicast   0          0
kind-worker          65200      65100     172.21.0.3     established     7m20s    ipv4/unicast   1          0
                                                                                  ipv6/unicast   0          0
```

Exec into the `kind-control-plane` Cilium Agent to verify the details of the advertised route:

```sh
$ kubectl get po -n kube-system -o wide | grep cil
cilium-zggj9    1/1     Running   0              145m   172.21.0.3    kind-control-plane   <none>           <none>
cilium-8tpzp    1/1     Running   0              145m   172.21.0.2    kind-worker          <none>           <none>
...

$ kubectl exec -it po/cilium-zggj9 -n kube-system -c cilium-agent -- cilium bgp routes available ipv4 unicast
VRouter   Prefix         NextHop   Age     Attrs
65100     10.10.1.0/24   0.0.0.0   5m53s   [{Origin: i} {Nexthop: 0.0.0.0}]
```

Do the same for the `kind-worker` node:

```sh
$ kubectl exec -it po/cilium-8tpzp -n kube-system -c cilium-agent -- cilium bgp routes available ipv4 unicast
VRouter   Prefix         NextHop      Age     Attrs
65200     10.10.1.0/24   172.21.0.3   8m35s   [{Origin: i} {AsPath: 65100} {Nexthop: 172.21.0.3}]
```

Note that the `172.21.0.3` next hop is the Internal IP of the `kind-control-plane` node.

## Advertise the Other Pool

Verify the `other` pool is not labled:

```sh
apiVersion: cilium.io/v2alpha1
kind: CiliumPodIPPool
metadata:
  creationTimestamp: "2023-09-22T18:15:51Z"
  generation: 1
  name: other
  resourceVersion: "900"
  uid: f68d1178-e0ee-4cf7-a557-5f4655286555
spec:
  ipv4:
    cidrs:
    - 10.20.0.0/16
    maskSize: 24
```

Add the `foo: bar` label to the `other` CiliumPodIPPool:

```sh
kubectl label ciliumpodippool/other foo=bar
```

Verify BGP CP routes:

```sh
$ cilium bgp peers
Node                 Local AS   Peer AS   Peer Address   Session State   Uptime   Family         Received   Advertised
kind-control-plane   65100      65200     172.21.0.2     established     7m20s    ipv4/unicast   0          1
                                                                                  ipv6/unicast   0          0
kind-worker          65200      65100     172.21.0.3     established     7m20s    ipv4/unicast   1          0
                                                                                  ipv6/unicast   0          0
```

Note that no addititional routes are being advertised. This is because no CIDRs from pool `other` have been
allocated to the `kind-control-plane` node:

```sh
$ kubectl get ciliumnode/kind-control-plane -o yaml
apiVersion: cilium.io/v2
kind: CiliumNode
...
  ipam:
    pools:
      allocated:
      - cidrs:
        - 10.10.1.0/24
        pool: default
```

Create a DaemonSet that uses pool `other`:

```sh
$ kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: netshoot
spec:
  selector:
    matchLabels:
      app: netshoot
  template:
    metadata:
      labels:
        app: netshoot
      annotations:
        ipam.cilium.io/ip-pool: other
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: netshoot
        image: nicolaka/netshoot:latest
        command: ["sleep", "infinite"]
EOF
```

Verify the DaemonSet pods are running and are assigned IP's from the `other` pool:

```sh
$ kubectl get po -o wide
NAME             READY   STATUS    RESTARTS   AGE   IP            NODE                 NOMINATED NODE   READINESS GATES
netshoot-kmx7j   1/1     Running   0          36s   10.20.0.112   kind-control-plane   <none>           <none>
netshoot-sl8hx   1/1     Running   0          36s   10.20.1.223   kind-worker          <none>           <none>
```

Verify the `kind-control-plane` node is allocated a CIDR from the `other` pool:

```sh
$ kubectl get ciliumnode/kind-control-plane -o yaml
apiVersion: cilium.io/v2
kind: CiliumNode
...
  ipam:
    pools:
      allocated:
      - cidrs:
        - 10.10.1.0/24
        pool: default
      - cidrs:
        - 10.20.0.0/24
        pool: other
```

You should now see `kind-control-plane` node advertise 2 routes: 

```sh
$ cilium bgp peers
Node                 Local AS   Peer AS   Peer Address   Session State   Uptime   Family         Received   Advertised
kind-control-plane   65100      65200     172.21.0.2     established     26m56s   ipv4/unicast   0          2
                                                                                  ipv6/unicast   0          0
kind-worker          65200      65100     172.21.0.3     established     26m56s   ipv4/unicast   2          0
                                                                                  ipv6/unicast   0          0
```

You can use the previous `cilium bgp routes available ipv4 unicast` commands to verify route details.

Lastly, you can update the `kind-worker` BGP peering policy to advertise Multi-Pool IPAM CIDRs.

Congratulations, you've completed the demo!

[mp-ipam]: https://docs.cilium.io/en/stable/network/kubernetes/ipam-multi-pool/
[mp-ipam-support]: https://github.com/cilium/cilium/pull/27100
