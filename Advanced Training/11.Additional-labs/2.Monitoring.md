# Calico Enterprise Monitoring and Alerting
This content includes the hands-on lab for Calico Enterprise monitoring and alerting using Prometheus and Prometheus Alertmanager.
All the commands in the following instructions are run from the bastion node unless otherwise specified in the instructions.

## Agenda

- Getting familiar With Prometheus deployment
- Viewing network and network policy metrics using Calico Enterprise Dashboard
- Viewing metrics in Prometheus
- Viewing alerts in Alertmanager

#### Relevant Docs

https://docs.tigera.io/maintenance/monitor/

### Getting Familiar With Prometheus Deployment

1. Run the following command to view the Prometheus pods that are running in the cluster. You should see the following pods.

  - Three Alertmanager pods.
  - One Prometheus operator pod.
  - One Prometheus server pod.
```
kubectl get pods -n tigera-prometheus
```
```
tigera@bastion:~$ kubectl get pods -n tigera-prometheus
NAME                                          READY   STATUS    RESTARTS   AGE
alertmanager-calico-node-alertmanager-0       2/2     Running   0          2d23h
alertmanager-calico-node-alertmanager-1       2/2     Running   0          2d23h
alertmanager-calico-node-alertmanager-2       2/2     Running   0          2d23h
calico-prometheus-operator-7dd764bcfd-gzp8j   1/1     Running   0          5d8h
prometheus-calico-node-prometheus-0           3/3     Running   1          5d8h
```
2. Run the following command to see all the Prometheus resources that are running in the cluster. You should see the following resources.
  - The pods that you viewed in the previous step.
  - Preconfigured services for connectivity to Alertmanager and Prometheus. Please note the two nodePorts for connectvity to Prometheus and Alertmanager are not preconfigured and were configured post Calico Enterprise install for the lab purpose. For production environments, it is recommended to use a more production ready solution such as an Ingress resource or Service IP advertisement for ingress connectivity to services.
  - A deployment for Prometheus operator.
  - A statefulset for Prometheus.
  - A statefulset for Alertmanager.

```
kubectl get all -n tigera-prometheus
```
```
tigera@bastion:~$ kubectl get all -n tigera-prometheus
NAME                                              READY   STATUS    RESTARTS   AGE
pod/alertmanager-calico-node-alertmanager-0       2/2     Running   0          2d23h
pod/alertmanager-calico-node-alertmanager-1       2/2     Running   0          2d23h
pod/alertmanager-calico-node-alertmanager-2       2/2     Running   0          2d23h
pod/calico-prometheus-operator-7dd764bcfd-gzp8j   1/1     Running   0          5d8h
pod/prometheus-calico-node-prometheus-0           3/3     Running   1          5d8h

NAME                                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/alertmanager-operated               ClusterIP   None            <none>        9093/TCP,9094/TCP,9094/UDP   5d8h
service/calico-node-alertmanager            ClusterIP   10.49.77.144    <none>        9093/TCP                     5d8h
service/calico-node-alertmanager-nodeport   NodePort    10.49.208.158   <none>        9093:30009/TCP               2d21h
service/calico-node-prometheus              ClusterIP   10.49.36.33     <none>        9090/TCP                     5d8h
service/calico-node-prometheus-nodeport     NodePort    10.49.29.80     <none>        9090:30006/TCP               3d1h
service/prometheus-operated                 ClusterIP   None            <none>        9090/TCP                     5d8h

NAME                                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/calico-prometheus-operator   1/1     1            1           5d8h

NAME                                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/calico-prometheus-operator-7dd764bcfd   1         1         1       5d8h

NAME                                                     READY   AGE
statefulset.apps/alertmanager-calico-node-alertmanager   3/3     5d8h
statefulset.apps/prometheus-calico-node-prometheus       1/1     5d8h
```

3. Run the following command to view the Prometheus custom resource configuration. Note the "ruleSelector:" section. These are the labels that Prometheus use to match Prometheus rules to be managed. Also note "serviceMonitorSelector:", which is used to match the servicemonitors to be managed by this instance of Prometheus.
###### Note: Some of the irrelevant  outpout below is removed.


```
tigera@bastion:~$ kubectl get -n tigera-prometheus prometheuses.monitoring.coreos.com -o yaml
apiVersion: v1
items:
- apiVersion: monitoring.coreos.com/v1
  kind: Prometheus
  metadata:
    name: calico-node-prometheus
    namespace: tigera-prometheus
  spec:
    alerting:
      alertmanagers:
      - name: calico-node-alertmanager
        namespace: tigera-prometheus
        port: web
        scheme: http
    baseImage: quay.io/prometheus/prometheus
    nodeSelector:
      kubernetes.io/os: linux
    podMonitorSelector:
      matchLabels:
        team: network-operators
    resources:
      requests:
        memory: 400Mi
    retention: 24h
    ruleSelector:
      matchLabels:
        prometheus: calico-node-prometheus
        role: tigera-prometheus-rules
    serviceAccountName: prometheus
    serviceMonitorSelector:
      matchLabels:
        team: network-operators
```

4. Run the following command to view the exisiting Prometheus rules configured in the cluster. Examine the rule by viewing the rule definition.

```
kubectl get -n tigera-prometheus prometheusrules.monitoring.coreos.com 
```

5.  Run the following command to view the servicemonitor configured to discover and scrape bgp and policy metrics. Note the followings in the service monitor definition:
  - Servicemonitor label is used to match the Prometheus resource
  - Seletor section  is used to match the calico-node-metric service.

```
kubectl get -n tigera-prometheus servicemonitors.monitoring.coreos.com calico-node-monitor -o yaml
```
```
tigera@bastion:~$ kubectl get -n tigera-prometheus servicemonitors.monitoring.coreos.com calico-node-monitor -o yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    team: network-operators
  name: calico-node-monitor
  namespace: tigera-prometheus
spec:
  endpoints:
  - honorLabels: true
    interval: 5s
    port: calico-metrics-port
    scrapeTimeout: 5s
  - honorLabels: true
    interval: 5s
    port: calico-bgp-metrics-port
    scrapeTimeout: 5s
  namespaceSelector:
    matchNames:
    - calico-system
  selector:
    matchLabels:
      k8s-app: calico-node
```


6. Run the followig command to view the service that is used to provide connectivity to bgp and policy Prometheus metrics.
###### Note: Some of the irrelevant  outpout below is removed.


```
kubectl get svc -n calico-system calico-node-metrics -o yaml
```
```
tigera@bastion:~$ kubectl get svc -n calico-system calico-node-metrics -o yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: calico-node
  name: calico-node-metrics
  namespace: calico-system
spec:
  clusterIP: 10.49.203.49
  clusterIPs:
  - 10.49.203.49
  ports:
  - name: calico-metrics-port
    port: 9081
    protocol: TCP
    targetPort: 9081
  - name: calico-bgp-metrics-port
    port: 9900
    protocol: TCP
    targetPort: 9900
  selector:
    k8s-app: calico-node
```


### Network and Network Policy Monitoring Using Calico Enterprise Dashboard

1. Use the provided token to log into Calico Enterprise Manager UI.

2. From the left navigation menu, select Dashboard. Get yourself familiar with the dashboard. There are various metrics such as ingress/egress connections, allowed/denied packets, allowed bytes and packets and others that are provided through Prometheus on the dasboard.

![tigeramanager-dashabord](https://github.com/tigera-cs/tigera-lab/blob/master/trainingworkbooks/Advanced%20Training/img/monitoring/TigeraManager-Dashboard.JPG)

3. In this section, we create a network policy and generate some allowd and denied traffic to view the metrics in Calico Enterprise Manager UI.

4. Run the following command and take note of the summary pod IP address.

```
kubectl get pods -n yaobank -l app=summary -o wide | head -n2
```
```
tigera@bastion:~$ kubectl get pods -n yaobank -l app=summary -o wide | head -n2
NAME                       READY   STATUS    RESTARTS   AGE    IP           NODE           NOMINATED NODE   READINESS GATES
summary-748b977d44-6vjt2   1/1     Running   0          3d1h   10.48.7.80   ip-10-0-1-30   <none>           <none>
```
5. Create the following network policy. This policy allows http and icmp traffic from from customer pod to summary pod.

```
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: default.allow-egress-tcp
  namespace: yaobank
spec:
  egress:
  - action: Allow
    destination:
      ports:
      - 80
    protocol: TCP
    source: {}
  - action: Allow
    destination:
      selector: app == "summary"
    protocol: ICMP
    source: {}
  selector: app == "customer"
  types:
  - Egress
EOF

```
6. Exec into the customer pod.

```
kubectl exec -it -n yaobank $(kubectl get pods -n yaobank -l app=customer -o name) -- bash
```

7. Run a curl against the ip address of the summary pod that we captured in the previous step. Please ensure to replace the "IP address" in the command below. Let the command run for 10 second and then ctrl+c. Immediately  go to the dashboard page in the Calico Enterprise Manager UI.

```
while true; do curl <IP Address>; done
```

8. View the allowed traffic in the "Packets by Policy" section. You should see the policy allowing the traffic along with the number of allowed packets. Examine other relevant metrics on the dashboard.

![tigeramanager-dashabord-allowedtraffic](https://github.com/tigera-cs/tigera-lab/blob/master/trainingworkbooks/Advanced%20Training/img/monitoring/Dashboard-AllowedTraffic.JPG)

9. Exec into the customer pod.

```
kubectl exec -it -n yaobank $(kubectl get pods -n yaobank -l app=customer -o name) -- bash
```

10. This time run a curl against the dns name of the summary pod. Let the command run for 10 second and then ctrl+c. Immediately  go to the dashboard page in the Calico Enterprise Manager UI.

```
while true; do curl summary.yaobank; done
```

![tigeramanager-dashabord-deniedtraffic](https://github.com/tigera-cs/tigera-lab/blob/master/trainingworkbooks/Advanced%20Training/img/monitoring/Dashboard-DeniedTraffic.JPG)


### Viewing metrics in Prometheus

1. Browse to the Prmetheus web console and from the Status menu select Configuration. Get yourself familiar with the configurations. Try to find the scrap settings for calico-node-monitor we viewed through servicemonitor in the previous excerise.

![pormetheus-config](https://github.com/tigera-cs/tigera-lab/blob/master/trainingworkbooks/Advanced%20Training/img/monitoring/Prometheus-Config.JPG)


2. Again follow the same previous steps, but this time from Status menu select Targets and view the available metrics endpoints and their statuses.


3. Browse to the Prometheus dashabord page and run the followwing query to check the status of calico-node pods. Note the value column. 

  - up

![prometheus-query-up.JPG](https://github.com/tigera-cs/tigera-lab/blob/master/trainingworkbooks/Advanced%20Training/img/monitoring/Prometheus-Query-Up.JPG)


4. Let's run a more granular query this time using the available metric labels.

  - up{endpoint="calico-metrics-port",instance="10.0.1.20:9081"}

![prometheus-query-up.JPG](https://github.com/tigera-cs/tigera-lab/blob/master/trainingworkbooks/Advanced%20Training/img/monitoring/Prometheus-Query-Up-Granular.JPG)

5. Let's run a query to check the bgp peers with established status for IPv4.

  - bgp_peers{status="Established", ip_version="IPv4"}

![prometheus-bgp-peer-established](https://github.com/tigera-cs/tigera-lab/blob/master/trainingworkbooks/Advanced%20Training/img/monitoring/prometheus-bgp-peer-established.JPG)


### Viewing alerts in Alertmanager

In this section, we will simulate some failue and we will check and Prometheus Alertmanager alerting capabilities.

1. Browse to the Prometheus web console and from the Status menu select Rules.

![prometheus-rules](https://github.com/tigera-cs/tigera-lab/blob/master/trainingworkbooks/Advanced%20Training/img/monitoring/prometheus-rules.JPG)

1. Create the following two Prometheus rules.
#### Note: We have set 5 seconds for the alerts to get trigerred to save on the lab time. Please make sure to set the right time in your production clusters.

```
kubectl apply -f -<<EOF
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:
    prometheus: calico-node-prometheus
    role: tigera-prometheus-rules
  name: tigera-prometheus-peer-status-not-established
  namespace: tigera-prometheus
spec:
  groups:
  - name: calico.rules
    rules:
    - alert: CalicoNodePeerStatusNotEstablished
      annotations:
        description: '{{$labels.instance}} has at least one peer connection that is
          no longer up.'
        summary: Instance {{$labels.instance}} has peer connection that is no longer
          up
      expr: rate(bgp_peers{status!~"Established"}[5s]) > 0
      labels:
        severity: critical
EOF
```

```
kubectl apply -f -<<EOF
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: calico-prometheus-calico-node-down
  namespace: tigera-prometheus
  labels:
    role: tigera-prometheus-rules
    prometheus: calico-node-prometheus
spec:
  groups:
  - name: calico.rules
    rules:
    - alert: CalicoNodeInstanceDown
      expr: up == 0
      for: 5s
      labels:
        severity: warning
      annotations:
        summary: "Instance {{$labels.instance}} Pod: {{$labels.pod}} is down"
        description: "{{$labels.instance}} of job {{$labels.job}} has been down for more than 5 seconds"
EOF
```
3. Browse to the Prometheus web console and from the Status menu select Rules again. Two Prometheus rules should be added. It might take a up to a minute for the rules to appear and the states get updated on the console.

4. Open two browser pages to Alertmanager console and Prometheus console (Alerts page).

![alertmanager-console](https://github.com/tigera-cs/tigera-lab/blob/master/trainingworkbooks/Advanced%20Training/img/monitoring/alertmanager-console.JPG)

![prometheus-alerts](https://github.com/tigera-cs/tigera-lab/blob/master/trainingworkbooks/Advanced%20Training/img/monitoring/prometheus-alerts.JPG)

6. Let's simulate a failure to see the alerts getting triggered. ssh into worker2 and run the following command and then wait for the alerts to get triggered.

```
ssh worker2
```
```
sudo reboot
```

5. Note the the triggered alerts. If you wait long enough for the rebooted node to come back online, you should see the alerts getting deactiavated.

![alertmanager-alerts-down](https://github.com/tigera-cs/tigera-lab/blob/master/trainingworkbooks/Advanced%20Training/img/monitoring/alertmanager-alerts-down.JPG)

![prometheus-alerts-down](https://github.com/tigera-cs/tigera-lab/blob/master/trainingworkbooks/Advanced%20Training/img/monitoring/prometheus-alerts-down.JPG)
