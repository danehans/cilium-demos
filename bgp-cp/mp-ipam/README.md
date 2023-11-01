# BGP Control Plane with Multi-Pool IPAM

This is a demonstration of using Cilium BGP Control Plane (BGP CP) to advertise
[Multi-Pool IPAM][mp-ipam] CIDRs to BGP peers. The demo setup is based on the Cilium
BGP CP [development environment][bgpcp-dev].

## Prerequisites
1. [kubectl](https://kubernetes.io/docs/tasks/tools/)
2. [kind](https://kind.sigs.k8s.io/)
3. [helm](https://helm.sh/docs/intro/install/)
4. [cilium-cli](https://github.com/cilium/cilium-cli) v0.15.11 or later.
5. [ContainerLab](https://containerlab.dev/install/) v0.45.1 or later
6. `git clone https://github.com/cilium/cilium.git && cd cilium`

## Create a Kubernetes Cluster

Use the `kind-bgp-v4` target to create a Kind cluster configured for IPv4:

```sh
make kind-bgp-v4
```

## Install Cilium

Install Cilium using the demo Helm values:

```sh
cilium install --chart-directory install/kubernetes/cilium -f https://raw.githubusercontent.com/danehans/cilium-demos/ciliumcon_na_2023/bgp-cp/mp-ipam/values.yaml
```

Wait for Cilium to report a "ready" status:

```sh
cilium status --wait
```

Cilium is installed and created two IPAM pools:

```sh
kubectl get cpip
```

However, only the `default` pool is allocated to the nodes. Verify the pool allocations on the
`control-plane` node:

```sh
kubectl get cn bgp-cplane-dev-v4-control-plane -o yaml
```

Verify the pool allocations on the `worker` node:

```sh
kubectl get cn bgp-cplane-dev-v4-worker -o yaml
```

Only the `default` pool is allocated because Cilium will only allocate an IPAM pool CIDR to a node
when requested by a Pod. We will create a Pod later in the demo that does this.

## Configure BGP CP

Create a BGP peering policy to peer the Cilium nodes with the external [FRRouting (FRR)][frr] router:

```sh
make kind-bgp-v4-apply-policy
```

Check the BGP peering status. The session state for each BGP speaker should be `established`.

```sh
cilium bgp peers
```

You can also check the peering status from the FRR router. First, exec into the FRR router:

```sh
docker exec -it clab-bgp-cplane-dev-v4-router0 vtysh
```

`vtysh` stands for "Virtual Terminal Shell" and it provides a unified interface to interact with
the various FRR daemons that manage different routing protocols.

From the vtysh, check the BGP status:

```sh
show ip bgp sum
```

No routes are being advertised due to the peering policy not being configured with a `podIPPoolSelector`.

Exit the FRR router and verify the peering policy configuration of the `control-plane` node:

```sh
kubectl get bgpp control-plane -o yaml
```

Verify the peering policy configuration of the `worker` node:

```sh
kubectl get bgpp worker -o yaml
```

Update the peering policy to advertise the IPAM CIDRs:

```sh
for i in control-plane worker; do
kubectl patch bgpp $i --type=json --patch '[{
   "op": "add",
   "path": "/spec/virtualRouters/0/podIPPoolSelector",
   "value": {
       "matchLabels": {
          "foo": "bar"
       },
    },
}]'; done
```

Note that the IPAM CIDRs are not labeled with `foo: bar` so routes no routes are being advertised:

```sh
cilium bgp peers
```

Verify that no IPAM pools contain the `foo: bar` label:

```sh
kubectl get cpip --show-labels
```

## Advertise the Default Pool

Add the `foo: bar` label to CiliumPodIPPool `default`:

```sh
kubectl label cpip default foo=bar
```

Verify BGP CP is now advertising the route:

```sh
cilium bgp peers
```

Exec into the `control-plane` node:

```sh
kubectl get po -n kube-system -o wide | grep cil
```

Exec into the `control-plane` node and verify the details of the advertised route (replacing `cilium-zggj9`
with the Cilium `control-plane` pod from the above output):

```sh
kubectl exec -it po/cilium-zggj9 -n kube-system -c cilium-agent -- cilium bgp routes available ipv4 unicast
```

Do the same for the `worker` node:

```sh
kubectl exec -it po/cilium-8tpzp -n kube-system -c cilium-agent -- cilium bgp routes available ipv4 unicast
```

Note that the NextHop address is the IP of the FRR router.

## Advertise the Other Pool

Verify the `other` pool is not labled:

```sh
kubectl get cpip --show-labels
```

Add the `foo: bar` label to the `other` CiliumPodIPPool:

```sh
kubectl label cpip other foo=bar
```

Verify BGP CP routes:

```sh
cilium bgp peers
```

Note that no addititional routes are being advertised. This is because no CIDRs from pool `other` have been
allocated to the nodes.

Verify that the `other` pool is not allocated to the `control-plane` node:

```sh
kubectl get cn bgp-cplane-dev-v4-control-plane -o yaml
```

Verify that the `other` pool is not allocated to the `worker` node:

```sh
kubectl get cn bgp-cplane-dev-v4-worker -o yaml
```

Create a DaemonSet that uses pool `other`:

```sh
kubectl apply -f - <<EOF
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

__Note:__ The `ipam.cilium.io/ip-pool` annotation is specified to request an IP from the
`other` IPAM pool.

Verify the DaemonSet is ready:

```sh
watch kubectl get ds netshoot
```

Verify the pod IP has been assigned from the `other` pool:

```sh
kubectl get po -o wide
```

__Note:__ Pods from the DaemonSet do not get scheduled to the `control-plane` node.

Verify the `worker` node was allocated a CIDR from the `other` pool:

```sh
kubectl get cn bgp-cplane-dev-v4-worker -o yaml
```

You should now see `worker` node advertise two routes:

```sh
cilium bgp peers
```

Get the IP of the netshoot pod:

```sh
$ kubectl get po -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP            NODE                       NOMINATED NODE   READINESS GATES
netshoot-r6x97          1/1     Running   0          72m   10.20.0.38    bgp-cplane-dev-v4-worker   <none>           <none>
...
```

Test connectivity from the external netshoot container > external router > the netshoot pod running
on the `worker` node:

```sh
docker exec -it clab-bgp-cplane-dev-v4-server2 mtr 10.20.0.38 -r
```

You can use the previous `cilium bgp routes available ipv4 unicast` commands to verify route details.

Congratulations, you've completed the demo!

[bgpcp-dev]: https://docs.cilium.io/en/latest/contributing/development/bgp_cplane/#development-environment
[frr]: https://frrouting.org/
[mp-ipam]: https://docs.cilium.io/en/stable/network/kubernetes/ipam-multi-pool/
