# 4. Network policies - Multitenancy

In this lab, you will define multi-tenancy rules for 2 tenants in your envionment.

Steps: \
4.1. Apply Calico policies to set up multi-tenancy \
4.2. Test

## 4.1. Apply Calico polcies to setup multi-tenancy

This lab, as brief as it is, is extremely important for understanding the role of Calico Enterprise Tiers and Pass action in establishing multi-tenancy. The design principle we are adopting here is one way of achieving fully micro-segmented environment. This is a common approach we have refined in engaging with customers to implement a standard design that can achieve the most complext requirements using a simple design.

```
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: security.pass-tenant1
spec:
  tier: security
  order: 101
  selector: tenant == "tenant1"
  types:
  - Ingress
  ingress:
  - action: Allow
    protocol: TCP
    destination:
      selector: ingress == "true"
      ports:
      - 80
  - action: Pass
    destination:
      selector: tenant == "tenant1"
---
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: security.pass-tenant2
spec:
  tier: security
  order: 102
  selector: tenant == "tenant2"
  types:
  - Ingress
  ingress:
  - action: Pass
    destination:
      selector: tenant == "tenant2"
---
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: security.default-deny-security-tier
spec:
  tier: security
  order: 999999
  selector: all()
  types:
  - Ingress
  ingress:
  - action: Deny
EOF
```

Notice the use of the broad scope scd , which matches all pods belonging to those tenants. We could have likewise used namespaceSelector == "tenant1", which can be a better approach if you think about it from a RBAC perspective, where the platform team manages namespace provisioning and labeling, while developers manage deployments to their authorised namespaces and pod labels.


What this is actually doing is delegating (passing) controls for tenant1 to the application tier after implementing high-level security controls at the security tier level. High level security controls are typically implemented through enterprise security guidelines and compliance requirements for intra-cluster and external communication. At the application tier level, granular microsegmentation would be implmentated by developers to secure microservices. 

It is important here to understand the policy processing behavior of Calico Enterprise:
- The default action for selected endpoint within a policy tier is Deny
- The default action for non-selected endpoint within a policy tier is Pass
- Pass action happens at the end of a policy tier

This means that for any selected endpoints in a policy tier, communication that is not explicitly permitted or passed is denied. This means that in the very simple policy we have implemented, we have effectively isolated tenant1 and tenant2. Communication for tenant1 for example outside the scope of its own tenant is denied since we're selecting tenant1 and only passing communication with its own tenant. Notice the default deny at the security tier effectively enforces organisation controls including multi-tenancy. Whatever is not passed or allowed is denied.


Take a moment and reflect on how it would have entailed to implement the same in a legacy infrastructure. The possibilities are simply unlimited!

## 4.2. Test

General security policies for the tenants will be controlled in the security tier, while anything not implicitily allowed will be passed to the next tier giving control to the team in charge of the application to apply more granular rules for microsegmentation. Now, as a developer, let's implement a simple policy to allow the ICMP communication within the microservices in app1 (we will see in the next lab how RBAC can segregate access, so a developer will be able to only create policies in his/her namespace).

```
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: application.allow-tenant1-icmp
spec:
  ingress:
  - action: Allow
    protocol: ICMP
  order: 10
  selector: tenant == "tenant1"
  tier: application
  types:
  - Ingress
EOF
```

Note the rule has been applied at the top of the application tier, before the yaobank rules.

Now, let's repeat the last test in our previous lab among services within the same namespace. Now the developer has allowed that comminication in the application tier, after any other higher hierarchy rule our security team has applied:

Let's again use one of the pods in the app1 namespace as our source pod for testing purposes.

```
APP1_POD=$(kubectl get pod -n app1 --no-headers -o name | head -1) && echo $APP1_POD
```

And let's get the IP address of the other pod in the namespace app1.

```
kubectl get pod -n app1 -o wide | tail -1
```
```
app1-deployment-5bbfd76f9d-mzjzd   1/1     Running   0          14m   10.48.0.212   ip-10-0-1-31.ca-central-1.compute.internal   <none>           <none>
```

Now, let's access our testing pod and try to ping to the other pod within the same namespace (substitute the IP address for the one retrieved in the previous step in your lab).

```
kubectl exec -ti $APP1_POD -n app1 -- sh
```
```
/ # ping 10.48.0.212
```

External communication must still fail
