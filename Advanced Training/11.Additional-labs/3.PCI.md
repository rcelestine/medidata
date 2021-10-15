<h1>PCI Audit Lab</h1>

In the following Lab we are going to prepare for PCI audits section 1.<br>
In this lab we will highlight where to find the relevant information. Notes that even if standards or compliance requirements are different they will often refer to similar documentation.<br>
First you need to ensure that log collections is enabled. create a policy file on the master node called policy.yaml under /home/ubuntu.<br>
```yaml
apiVersion: audit.k8s.io/v1beta1
kind: Policy
omitStages:
- RequestReceived
rules:
  - level: RequestResponse
    verbs:
      - create
      - patch
      - update
      - delete
    resources:
    - group: networking.k8s.io
      resources: ["networkpolicies"]
    - group: extensions
      resources: ["networkpolicies"]
    - group: ""
      resources: ["pods", "namespaces", "serviceaccounts", "endpoints"]
```
<br> Then under /etc/kubernetes/manifest/ edit your kube-apiserver.yaml and add the following under the appropriate section. Note that it will result in a short unavailability of the kube-apiser until the static pod is recreated.<br>
```yaml
# Add the flags below respecting indentation under - kube-apiserver

    - --audit-log-path=/var/log/calico/audit/kube-audit.log
    - --audit-policy-file=/home/ubuntu/policy.yaml

# Add the mounthPath below under volumeMounts

    - mountPath: /home/ubuntu
      name: audit
      readOnly: true
    - mountPath: /var/log/calico/audit/
      name: audit-logs

# Add the hostPath below under volumes

    - hostPath:
        path: /home/ubuntu
        type: DirectoryOrCreate
      name: audit
    - hostPath:
        path: /var/log/calico/audit/
        type: DirectoryOrCreate
      name: audit-logs
```


We are going to start with enforcing the first requirements of PCI which is around endpoint control.<br>
![PCI requierements](https://github.com/tigera-cs/tigera-lab/blob/master/trainingworkbooks/Advanced%20Training/img/pci/img1.png)




We will be using the yaobank application as the application requiring PCI compliance and apply the guidance below to pods in this namespace.

First assign a PCI label to each of the pods. (note that label should be defined in deployment/replica set/ static pod however for the purpose of this lab this will be done manually.)

```kubectl get pods -n yaobank```

Copy the name of the pod and apply a label to 2 of them as per the example below:<br>
```kubectl label pods customerxyz -n yaobank pci=ok```<br>
```kubectl label pods summaryxyz -n yaobank pci=ok```

Do the same for the 2 matching yaobank services.  
```kubectl label svc customerxyz -n yaobank pci=ok ```<br>
```kubectl label svc summaryxyz -n yaobank pci=ok```

Double check the label is correctly applied to each pod and services.  
```kubectl get pods -n yaobank --show-labels```<br>
```kubectl get svc -n yaobank --show-labels```<br>
Then we will apply policies to:
	Block all traffic with non-PCI Workload
	Allow traffic across PCI workloads
```yaml
cat <<EOF | kubectl create -f -
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default.pci
spec:
  tier: default
  order: 0
  selector: pci == "ok"
  namespaceSelector: ''
  serviceAccountSelector: ''
  ingress:
    - action: Allow
      source:
        selector: pci == "ok"
      destination: {}
    - action: Deny
      source:
        selector: pci != "ok"
      destination: {}
  egress:
    - action: Allow
      source: {}
      destination:
        selector: pci == "ok"
    - action: Deny
      source: {}
      destination:
        selector: pci != "ok"
  doNotTrack: false
  applyOnForward: false
  preDNAT: false
  types:
    - Ingress
    - Egress
EOF
```

Go under your policy tab and check that it does reflect that 2 endpoints are being protected.<br>
![PCI policy](https://github.com/tigera-cs/tigera-lab/blob/master/trainingworkbooks/Advanced%20Training/img/pci/img2.png)

With the endpoint secured and traffic controlled let’s create a report so we can monitor and provide proof that we are compliant with PCI on the first set of criterias.
Important to note that it will take 24 hours before policy are reflected on the dashboard<br>
![PCI requierements audit](https://github.com/tigera-cs/tigera-lab/blob/master/trainingworkbooks/Advanced%20Training/img/pci/img3.png)



The following will take you through the steps needed to create 2 manual Inventory Reports, leveraging the Compliance Reports module.

First, let’s ensure that the compliance-benchmarker is running, and that the Inventory report type is installed:

```kubectl get -n tigera-compliance daemonset compliance-benchmarker```<br>
```kubectl get globalreporttype inventory```
Next we will create 2 reports scheduled to run hourly using the inventory template.
The first one for all endpoints using the namespace names: yaobank
```yaml
apiVersion: projectcalico.org/v3
kind: GlobalReport
metadata:
  name: hourly-yaobank-inventory
spec:
  reportType: inventory
  endpoints:
    namespaces:
      names: ["yaobank"]
  schedule: 7 * * * *
  ```
<br>The second one to report on all endpoints set with a pci == ok label.<br>
```yaml
apiVersion: projectcalico.org/v3
kind: GlobalReport
metadata:
  name: hourly-yaobank-inventory-pci
spec:
  reportType: inventory
  endpoints:
    selector: pci == 'ok'
  schedule: 37 * * * *
  ```
You can view the status of a report, you must use the kubectl command.  
```kubectl get globalreports.projectcalico.org hourly-yaobank-inventory -o yaml```<br>
Take a note of the time format used.

Change the default report generation time
By default, reports are generated 30 minutes after the end of the report, to ensure all of the audit data is archived. (However, this gap does not affect the data collected “start/end time” for a report.)
You can adjust the time for audit data for cases like initial report testing, to demo a report, or when manually creating a report that is not counted in global report status.
To change the delay, go to the installation manifest, and uncomment and set the environment TIGERA_COMPLIANCE_JOB_START_DELAY
We will now run a manual Inventory report.
Edit the template below as follows:
Edit the pod name if required.
If you are using your own docker repository, update the container image name with your repo and image tag.
Set the following environments according to the instructions in the downloaded manifest:
Remember to use UTC format for the time.
Apply the updated manifest, and query the status of the pod to ensure it completes.
 
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: run-reporter
  namespace: tigera-compliance
  labels:
    k8s-app: compliance-reporter
spec:
  nodeSelector:
    kubernetes.io/os: linux
  restartPolicy: Never
  serviceAccount: tigera-compliance-reporter
  serviceAccountName: tigera-compliance-reporter
  tolerations:
    - key: node-role.kubernetes.io/master
      effect: NoSchedule
  imagePullSecrets:
    - name: tigera-pull-secret
  containers:
  - name: reporter
    # Modify this image name, if you have re-tagged the image and are using a local
    # docker image repository.
    # ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    image: quay.io/tigera/compliance-reporter:v3.5.2
    # ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    env:
      # Modify this value with name of an existing globalreport resource.
      # ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    - name: TIGERA_COMPLIANCE_REPORT_NAME
      value: hourly-yaobank-inventory
      # Modify these values with the start and end time frame that should be reported on.
      # ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    - name: TIGERA_COMPLIANCE_REPORT_START_TIME
      value: 2021-04-23T14:20:00Z
    - name: TIGERA_COMPLIANCE_REPORT_END_TIME
      value: 2021-04-23T14:22:00Z
      # ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    - name: LOG_LEVEL
      value: "warning"
    - name: ELASTIC_INDEX_SUFFIX
      value: cluster
    - name: ELASTIC_SCHEME
      value: https
    - name: ELASTIC_HOST
      value: tigera-secure-es-http.tigera-elasticsearch.svc
    - name: ELASTIC_PORT
      value: "9200"
    - name: ELASTIC_USER
      valueFrom:
        secretKeyRef:
          name: tigera-ee-compliance-reporter-elasticsearch-access
          key: username
          optional: true
    - name: ELASTIC_PASSWORD
      valueFrom:
        secretKeyRef:
          name: tigera-ee-compliance-reporter-elasticsearch-access
          key: password
          optional: true
    - name: ELASTIC_SSL_VERIFY
      value: "true"
    - name: ELASTIC_CA
      value: /etc/ssl/elastic/ca.pem
    volumeMounts:
    - mountPath: /var/log/calico
      name: var-log-calico
    - name: elastic-ca-cert-volume
      mountPath: /etc/ssl/elastic/
    livenessProbe:
      httpGet:
        path: /liveness
        port: 9099
        host: localhost
  volumes:
  - name: var-log-calico 
    hostPath:
      path: /var/log/calico
      type: DirectoryOrCreate
  - name: elastic-ca-cert-volume
    secret:
      optional: true
      items:
      - key: tls.crt
        path: ca.pem
      secretName: tigera-secure-es-http-certs-public
```
Apply the compliance-reporter-pod.yaml
You can run another manual report by duplicating the manifest above, give the pod a different name and select the other inventory report (hourly-yaobank-inventory-pci)

Check the pod status:  
```kubectl get pods -n tigera-compliance | grep run-reporter```<br>
```run-reporter                              0/1     Completed   0          63s```<br>

Upon completion, the report is available in Calico Enterprise Manager.

Under the compliance tab in the manager-ui the following status should be displayed: 

The first one is our inventory report on the Yaobank namespace, we can see that only 2 endpoints are protected as the pci == ok label has been applied to only 2 pods.<br>
![PCI report](https://github.com/tigera-cs/tigera-lab/blob/master/trainingworkbooks/Advanced%20Training/img/pci/img4.png)

The second inventory report on pci == ok label shows that all endpoint with the label are protected however only 2 endpoints are counted as expected.<br>
![PCI report 2](https://github.com/tigera-cs/tigera-lab/blob/master/trainingworkbooks/Advanced%20Training/img/pci/img5.png)

If you want to implement further change you can change label on pod. <br>
```kubectl label pods database-784bc6b59f-mh4fh -n yaobank pci=ok --overwrite```<br>

Validate change:<br>
```kubectl get pods -n yaobank --show-labels```<br>
Check the changes.<br>
![PCI policy updated](https://github.com/tigera-cs/tigera-lab/blob/master/trainingworkbooks/Advanced%20Training/img/pci/img6.png)



Delete pod used to run previous manual report:<br>
```kubectl delete pod run-reporter -n=tigera-compliances ```<br>
Update time values so it’s ready to run in the next minutes and apply
```kubectl apply -f compliance-reporter.yaml```<br>


