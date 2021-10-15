# 7. Protecting non-cluster nodes with Calico Policy

This lab will demonstrate how you can use Calico to protect hosts that are not kubernetes cluster nodes

In this lab, you will: \
7.1. Install calico on the bastion host and configure it with failsafes so that you will not lock yourself out by accident\
7.2. Install a webserver that will be used as the target service \
7.3. Enable Automatic HostEndpoints for your kubernetes cluster nodes \
7.4. Configure a GlobalNetworkPolicy to allow traffic to the webserver \
7.5. Configure a HostEndpoint for the bastion host that will start to enforce the traffic

## 7.1. Install Calico on the bastion host

Install ipset on the bastion host as it is required by Calico 

```
sudo apt-get install -y ipset
```

Extract the cnx-node binary from the control1 node and transfer it to the bastion node

```
ssh control1
```
```
sudo docker create --name container quay.io/tigera/cnx-node:v3.7.0
sudo docker cp container:/bin/calico-node cnx-node
sudo docker rm container
```

Get back to the bastion host, and prepare the binary

```
exit
```
```
scp control1:cnx-node .
sudo mv cnx-node /usr/local/bin/
sudo chown root.root /usr/local/bin/cnx-node
sudo chmod 755 /usr/local/bin/cnx-node
```

Calico will be started with a configuration file /etc/calico/calico.env from a systemd unit file

```
sudo mkdir /etc/calico
```
```
sudo bash -c 'cat << EOF > /etc/calico/calico.env      
FELIX_DATASTORETYPE=kubernetes
KUBECONFIG=/home/tigera/.kube/config
CALICO_NETWORKING_BACKEND=none
FELIX_FAILSAFEINBOUNDHOSTPORTS="tcp:0.0.0.0/0:22,udp:0.0.0.0/0:68,udp:0.0.0.0/0:53,tcp:0.0.0.0/0:179"
FELIX_FAILSAFEOUTBOUNDHOSTPORTS="tcp:0.0.0.0/0:22,udp:0.0.0.0/0:53,udp:0.0.0.0/0:67,tcp:0.0.0.0/0:179,tcp:0.0.0.0/0:6443"
EOF'
```
```
sudo bash -c 'cat << EOF > /etc/systemd/system/calico.service
[Unit]
Description=Calico Felix agent
After=syslog.target network.target

[Service]
User=root
EnvironmentFile=/etc/calico/calico.env
ExecStartPre=/usr/bin/mkdir -p /var/run/calico
ExecStart=/usr/local/bin/cnx-node -felix
KillMode=process
Restart=on-failure
LimitNOFILE=32000

[Install]
WantedBy=multi-user.target
EOF'
```

Reload systemd daemon, enable and start Calico

```
sudo systemctl daemon-reload
sudo systemctl enable calico
sudo systemctl start calico
```

If you execute `sudo systemctl status calico` in the bastion host, the service should be now in a running state.

## 7.2. Emulate a running service

We will use netcat to simulate an application listening on port 7777 on the bastion node. Open a second tab to the lab (`http://<LABNAME>.lynx.tigera.ca`), and once logged in, execute the following command.

```
netcat -nvlkp 7777
```

Now get back to the original tab where you were working and test the connectivity to the server from your master node:

```
ssh control1
```
```
nc -zv 10.0.1.10 7777
```
```
Connection to 10.0.1.10 7777 port [tcp/*] succeeded!
```

Exit the master node, but keep running the service in your second tab as we will verify the connectivity again after implementing our Host Endpoint configuration.

## 7.3. Enable Automatic HostEndpoints for your kubernetes cluster nodes

Enable Automatic HostEndpoints by patching kubecontrollersconfiguration. 
We will use the automatic HostEndpoints from workers as the source in our GlobalNetworkPolicy that will protect the bastion node.

```
kubectl patch kubecontrollersconfiguration default --patch='{"spec": {"controllers": {"node": {"hostEndpoint": {"autoCreate": "Enabled"}}}}}'
```

## 7.4. Configure a GlobalNetworkPolicy to allow traffic to the webserver

Check the following globalnetworkpolicy. Notice how we select the bastion node by label type == 'bastion' or bastion = 'true'. This corresponds to the HostEndpoint we'll apply in the next step. Also notice how we allow traffic to port 7777 only from kubernetes nodes that have the label node-role.kubernetes.io/worker and allow traffic to port 80 from any source (this is needed to avoid lock up ourselves, as we access the lab through ttyd on that port).

```
kubectl get node -o=custom-columns=NAME:.metadata.name,LABELS:.metadata.labels
```
```
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: platform.bastionfirewall
spec:
  egress:
  - action: Allow
    destination:
      ports:
      - 80
      - 443
    protocol: TCP
  ingress:
  - action: Allow
    destination:
      ports:
      - 7777
    protocol: TCP
    source:
      selector: has(node-role.kubernetes.io/worker)||egress-code == "red"||app == "app1"
  - action: Allow
    destination:
      ports:
      - 80
    protocol: TCP
    source: {}
  selector: type == "bastion"||bastion == "true"
  tier: platform
  types:
  - Ingress
  - Egress
EOF
```


Apply the HostEndpoint. Note that manual HostEndpoints are default deny. Anything that is not allowed by failsafes or GlobalNetworkPolicies will be denied.

```
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: HostEndpoint
metadata:
  name: bastion
  labels:
    bastion: "true"
spec:
  interfaceName: "ens5"
  node: bastion
  expectedIPs:
  - 10.0.1.10
EOF
```

Test that traffic is being allowed only from a worker node

```
ssh worker1
```
```
nc -zv 10.0.1.10 7777
```
```
Connection to 10.0.1.10 7777 port [tcp/*] succeeded!
```

Try now from the control node, this test must fail:

```
ssh control1
```
```
nc -zv 10.0.1.10 7777
```

You can leave this second tab open, as we will use it in our next lab.
