# 8. Egress Gateway

## 8.0. Introduction

The purpose of this lab is to demonstrate the funciationality of Egress Gateway. In this lab, we will:

8.1. Enable per namespace Egress Gateway support \
8.2. Deploy an Egress Gateway in *one* namespace \
8.3. Enable BGP with upstream routers (Bird) and advertise the Egress Gateway IP Pool \
8.4. Test and Verify the communication

## 8.1. Enable egress gateway support on a per-namespace basis

Patch felix configuration to support per namespace egress gateway support

```
kubectl patch felixconfiguration.p default --type='merge' -p \
    '{"spec":{"egressIPSupport":"EnabledPerNamespace"}}'
```

## 8.1.1. Create IPPool

Create an IP Pool for Egress Gateway Pod

```
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: egress-ippool-1
spec:
  cidr: 10.50.0.0/31
  blockSize: 31
  nodeSelector: "!all()"
EOF
```

## 8.1.2. Apply Calico Enterprise pull secret to egress gateway namespace:

Create a pull secret into egress gateway namespace

```
kubectl create secret generic egress-pull-secret \
  --from-file=.dockerconfigjson=/home/tigera/config.json \
  --type=kubernetes.io/dockerconfigjson -n app1
```

## 8.2. Deploy Egress Gateway

### 8.2.1.EGW

Deploy the Egress gateway with the desired label to be used as an selector by namespace and app workloads using this Egress Gateway. In this example, the label we are using is `egress-code: red`. Please also note the IP Pool assigned to this Egress Gateway. In the cni.projectcalico.org/ipv4pools annotation, the IP Pool can be specified either by its name (e.g. egress-ippool-1) or by its CIDR (e.g. 10.58.0.0/31).


```
kubectl apply -f -<<EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: egress-gateway
  namespace: app1
  labels:
    egress-code: red
spec:
  selector:
    matchLabels:
      egress-code: red
  template:
    metadata:
      annotations:
        cni.projectcalico.org/ipv4pools: "[\"10.50.0.0/31\"]"
      labels:
        egress-code: red
    spec:
      imagePullSecrets:
      - name: egress-pull-secret
      nodeSelector:
        kubernetes.io/os: linux
      containers:
      - name: egress-gateway
        image: quay.io/tigera/egress-gateway:v3.7.0
        env:
        - name: EGRESS_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        securityContext:
          privileged: true
      terminationGracePeriodSeconds: 0
EOF
```

## 8.2.2. Connect the namespace to the gateways it should use

```
kubectl annotate ns app1 egress.projectcalico.org/selector='egress-code == "red"'
```

### 8.2.3. Verify both, the POD and Egress Gateway

```
kubectl get pods -n app1 -o wide 
```
```
tigera@bastion:~$ kubectl get pods -n app1 -o wide 
NAME                               READY   STATUS    RESTARTS   AGE   IP            NODE                                         NOMINATED NODE   READINESS GATES
app1-deployment-5bbfd76f9d-2zrdt   1/1     Running   0          7d    10.48.0.80    ip-10-0-1-30.ca-central-1.compute.internal   <none>           <none>
app1-deployment-5bbfd76f9d-pxlx8   1/1     Running   0          7d    10.48.0.203   ip-10-0-1-31.ca-central-1.compute.internal   <none>           <none>
egress-gateway-jc4wm               1/1     Running   0          40s   10.50.0.1     ip-10-0-1-30.ca-central-1.compute.internal   <none>           <none>
egress-gateway-n2zh2               1/1     Running   0          40s   10.50.0.0     ip-10-0-1-31.ca-central-1.compute.internal   <none>           <none>
```

## 8.3. BGP

### 8.3.1. BGP configuration on Calico

Deploy the needed BGP config, so we route our traffic to the bastion host through the egress gateway:

```
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgppeer-global-64512
spec:
  peerIP: 10.0.1.10
  asNumber: 64512
---
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  serviceClusterIPs:
  - cidr: 10.49.0.0/16
  communities:
  - name: bgp-large-community
    value: 64512:120
  prefixAdvertisements:
  - cidr: 10.50.0.0/31
    communities:
    - bgp-large-community
    - 64512:120
EOF
```

### 8.3.2. Check Calico Nodes connect to the bastion host

Our bastion host is simulating our ToR switch, and it should have BGP sessions established to all nodes:

```
sudo birdc show protocols
```

```
BIRD 1.6.8 ready.
name     proto    table    state  since       info
direct1  Direct   master   up     08:15:34    
kernel1  Kernel   master   up     08:15:34    
device1  Device   master   up     08:15:34    
control1 BGP      master   up     13:24:20    Established   
worker1  BGP      master   up     13:24:20    Established   
worker2  BGP      master   up     13:24:20    Established   
```

If you check the routes, you will see the edge gateway is reachable through the worker node where it has been deployed:

```
ip route
```
```
tigera@bastion:/var/run/bird$ ip route
default via 10.0.1.1 dev ens5 proto dhcp src 10.0.1.10 metric 100 
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.10 
10.0.1.1 dev ens5 proto dhcp scope link src 10.0.1.10 metric 100 
10.49.0.0/16 proto bird 
        nexthop via 10.0.1.20 dev ens5 weight 1 
        nexthop via 10.0.1.30 dev ens5 weight 1 
        nexthop via 10.0.1.31 dev ens5 weight 1 
10.50.0.0/31 via 10.0.1.31 dev ens5 proto bird 
10.50.0.1 via 10.0.1.30 dev ens5 proto bird 
```

## 8.4. Verification

### 8.4.1. Test and verify Egress Gateway in action

Open a second browser to your lab (`<LABNAME>.lynx.tigera.ca`) if not already done so that we have an additional terminal to test the egress gateway.

On that terminal, start netcat in the bastion host to listen to an specific port:

```
netcat -nvlkp 7777
```

On the original terminal window, exec into any of the pods in the app1 namespace.

```
APP1_POD=$(kubectl get pod -n app1 --no-headers -o name | head -1) && echo $APP1_POD
```
```
kubectl exec -ti $APP1_POD -n app1 -- sh
```

And try to connect to the port in the bastion host.

```
nc -zv 10.0.1.10 7777
```

Type `exit` to exit out the pod terminal.

Go to the terminal that you ran the following netcat server on the bastion node. You should see an output saying you connected from the IP of one of the egress gateway pods to the netcat server.

```
$ sudo tcpdump -i ens5 port 7777
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ens5, link-type EN10MB (Ethernet), capture size 262144 bytes
07:51:50.709295 IP ip-10-50-0-0.ca-central-1.compute.internal.41521 > ip-10-0-1-10.ca-central-1.compute.internal.7777: Flags [S], seq 1965935313, win 62727, options [mss 8961,sackOK,TS val 4198299208 ecr 0,nop,wscale 7], length 0
07:51:50.709346 IP ip-10-0-1-10.ca-central-1.compute.internal.7777 > ip-10-50-0-0.ca-central-1.compute.internal.41521: Flags [R.], seq 0, ack 1965935314, win 0, length 0
```

Stop the netcat listener process in the bastion host with `^C`.

### 8.4.2. Routing info on the Calico Node where App Workload is running

Login to worker node where the egress gateway and pods were deployed:

```
ssh worker2
```

Observe the routing policy is programmed for the the App workload POD IP

```
ip rule
```

```
0:      from all lookup local
100:    from 10.48.116.155 fwmark 0x80000/0x80000 lookup 250
32766:  from all lookup main
32767:  from all lookup default
```

Confirm that the policy is choosing the egress gateway as the next hop for any source traffic from App workload POD IP. 
#### Note: ensure to use the correct table number with the following command. In this case, the table number is 250.

```
ip route show table 250
```

```
default onlink 
        nexthop via 10.50.0.0 dev egress.calico weight 1 onlink 
        nexthop via 10.50.0.1 dev egress.calico weight 1 onlink
```

You can close the second browser tab with the terminal now as we will not use it for the rest of the labs.

## Conclusion

In this lab we observed how we can successfully setup egress gateway in a calico cluster and share routing information with external routing device over BGP.

