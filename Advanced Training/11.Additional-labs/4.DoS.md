<h1>Denail of Service Lab</h1>

In the following lab we will see how we can leverage anomalous flow detection and implement global policies to protect our environment.

First let’s create some anomaly detection jobs.

```curl https://docs.tigera.io/manifests/threatdef/ad-jobs-deployment.yaml -O```
<br>
```vi ad-jobs-deployment.yaml```

Update environmental values with the one of interest:


          - name: AD_train_interval_minutes 
            value: "1440"

We update training interval

          - name: AD_search_interval_minutes
            value: "30"

We update interval between scans

          - name: AD_ip_sweep_threshold
            value: "15"


We use IP sweep as given the nature of the lab and data trained, this is the job that is most likely to trigger a reaction in this sandbox environment.

Check logs and follow progress on learning.
```kubectl logs ad-jobs-deployment-8c95b8879-ss7ph -n tigera-intrusion-detection```

Create an alert:
```yaml
apiVersion: projectcalico.org/v3
kind: GlobalAlert
metadata:
  name: ping-flood-alert
spec:
  description: "25 ping Example"
  summary: "Ping example ${count} > 25"
  severity: 75
  dataSet: flows
  query: action='allow' AND proto='icmp'
  aggregateBy: [source_name_aggr]
  metric: count
  condition: gte
  threshold: 25
  ```

Check on existing Global alerts:<br>
```kubectl get globalalerts```<br>

You can also use the Calico UI and check the alerts through the interface<br>
Under alter, click on the configuration on the top right corner<br>
 
![Alerts](https://github.com/tigera-cs/tigera-lab/blob/master/trainingworkbooks/Advanced%20Training/img/DoS/img1.png)

Deploy nmap to run a IP scan<br>.

```kubectl run -i --tty netutils --image=dersimn/netutils -n yaobank -- bash```<br>
```nmap -sP 192.168.71.1-255```<br>






Validate alert on dashboard<br>
![Dashboard](https://github.com/tigera-cs/tigera-lab/blob/master/trainingworkbooks/Advanced%20Training/img/DoS/img2.png)


Apply a policy to stop ping:
```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: block-icmp
  namespace: yaobank
spec:
  order: 200
  selector:
  types:
  - Ingress
  - Egress
  ingress:
  - action: Deny
    protocol: ICMP
  - action: Deny
    protocol: ICMPv6
  egress:
  - action: Deny
    protocol: ICMP
  - action: Deny
    protocol: ICMPv6
```

Run the nmap command again, over denied ping and retry it will take a lot longer to run, pressing enter will return estimated time.

Check alerts
