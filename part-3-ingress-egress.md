# Controlling Ingress and Egress Traffic with Network Policy

The Kubernetes NetworkPolicy API allows users to express ingress and egress policies.  This portion of the lab walks through using Kubernetes NetworkPolicy to define more complex network policies.

> Controlling Egress is enhanced by using the **calicoctl** interface (not included in this lab).  To get the most fined grained control over traffic ingress-ing and egressing (especially) your cluster, Istio service mesh provides a fantastic solution.

Tutorial Flow:

* Create the Namespace and Nginx Service
* Deny all ingress traffic
* Allow ingress traffic to Nginx
* Deny all egress traffic
* Allow egress traffic to kube-dns

### Create the Namespace and Nginx Service

We’ll use a new namespace for this guide. Run the following commands to create it and a plain nginx service listening on port 80.

```
kubectl create ns advanced-policy-demo
kubectl run --namespace=advanced-policy-demo nginx --replicas=2 --image=nginx
kubectl expose --namespace=advanced-policy-demo deployment nginx --port=80
```

#### Verify Access - Allowed All Ingress and Egress

If you do not still have your busybox shell available, open up a second shell session which has kubectl connectivity to the Kubernetes cluster and create a busybox pod to test policy access. This pod will be used throughout this tutorial to test policy access.

```
kubectl run --namespace=advanced-policy-demo access --rm -ti --image busybox /bin/sh
```

This will open up a shell session inside the access pod, as shown below.

```
Waiting for pod advanced-policy-demo/access-472357175-y0m47 to be running, status is Pending, pod ready: false
If you don't see a command prompt, try pressing enter.
/ #
```

Now from within the busybox **access** pod execute the following command to test access to the nginx service.

```
wget -q --timeout=5 nginx -O -
```

It should return the HTML of the nginx welcome page.

Still within the busybox **access** pod, issue the following command to test access to google.com.

```
wget -q --timeout=5 google.com -O -
```

It should return the HTML of the google.com home page.  This is pretty, but it shows that your pod has egressed to Google and returned a payload.

### Deny all ingress traffic

Enable ingress isolation on the namespace by deploying a default deny all ingress traffic policy.

```
kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: advanced-policy-demo
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
  - Ingress
EOF
```

#### Verify Access - Denied All Ingress and Allowed All Egress

Since all pods in the namespace are now selected, any ingress traffic which is not explicitly allowed by a policy will be denied.

We can see that this is the case by switching over to our **access** pod in the namespace and attempting to access the nginx service.

```
wget -q --timeout=5 nginx -O -
```

It should return:

```
wget: download timed out
```

Next, try to access google.com.

```
wget -q --timeout=5 google.com -O -
```

It should return:

```
<!doctype html><html itemscope="" item....
```

We can see that the ingress access to the nginx service is denied while egress access to outbound internet is still allowed.

### Allow Ingress Traffic to Nginx

Run the following to create a NetworkPolicy which allows traffic to nginx pods from any pods in the **advanced-policy-demo** namespace.

```
kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-nginx
  namespace: advanced-policy-demo
spec:
  podSelector:
    matchLabels:
      run: nginx
  ingress:
    - from:
      - podSelector:
          matchLabels: {}
EOF
```

#### Verify Access - Allowed Nginx Ingress

Now ingress traffic to nginx will be allowed. We can see that this is the case by switching over to our **access** pod in the namespace and attempting to access the nginx service.

```
wget -q --timeout=5 nginx -O -
```

It should return:

```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>...
```

After creating the policy, we can now access the nginx Service.

### Deny All Egress Traffic

Configure egress isolation on the namespace by deploying a default deny all egress traffic policy.

> Within your enterprise environments for instance, you would not want your frontend pods that receive traffic through your proxies to be able to directly traverse into your trusted datacenter network.  You would like pass this off to a separate backend service to perform the trusted egress connection.

```
kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: advanced-policy-demo
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
  - Egress
EOF
```

#### Verify Access - Denied All Egress

Now any ingress or egress traffic which is not explicitly allowed by a policy will be denied.

We can see that this is the case by switching over to our **access** pod in the namespace and attempting to nslookup nginx or wget google.com.

```
nslookup nginx
```

It should return something like the following.

```
;; connection timed out; no servers could be reached
```

Next, try to access google.com.

```
wget -q --timeout=5 google.com -O -
```

It should return:

```
wget: bad address 'google.com'
```

> The nslookup command can take a minute or more to timeout.

### Allow DNS Egress Traffic

Run the following to create a label of name: kube-system on the kube-system namespace and a NetworkPolicy which allows DNS egress traffic from any pods in the advanced-policy-demo namespace to the kube-system namespace.

> A minor difference between Kuberenetes NetworkPolicy and Calico Policy set using **calicoctl** is **calicoctl** allows you to write policty directly to the namespace object where **kubectl** NetworkPolicy requires a selector running against a label you create for the namespace.

```
kubectl label namespace kube-system name=kube-system
kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-access
  namespace: advanced-policy-demo
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53

EOF
```

#### Verify Access - Allowed DNS access

Next, egress traffic to DNS will be allowed.  We can see that this is the case by switching over to our **access** pod in the namespace and attempting to lookup nginx and google.com.

```
nslookup nginx
```

It should return something like the following.

```
Server:		10.0.0.10
Address:	10.0.0.10:53 

Name:	nginx.advanced-policy-demo.svc.cluster.local
Address: 10.0.0.13

Can't find nginx.svc.cluster.local: No answer
Can't find nginx.cluster.local: No answer
Can't find nginx.ibmcloud.com: No answer
Can't find nginx.advanced-policy-demo.svc.cluster.local: No answer
Can't find nginx.svc.cluster.local: No answer
```

Next, try to look up google.com.

```
nslookup google.com
```

It should return something like the following.

```
Server:		10.0.0.10
Address:	10.0.0.10:53

Non-authoritative answer:
Name:	google.com
Address: 2607:f8b0:4000:806::200e

Can't find google.com: No answer
```

Even though DNS egress traffic is now working, all other egress traffic from all pods in the advanced-policy-demo namespace is still blocked. Therefore the HTTP egress traffic from the wget calls will still fail.

### Allow egress from pods in the advanced-policy-demo namespace to nginx

Run the following to create a NetworkPolicy which allows egress traffic from any pods in the advanced-policy-demo namespace to pods with labels matching run: nginx in the same namespace.

```
kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-to-advance-policy-ns
  namespace: advanced-policy-demo
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          run: nginx
EOF
```

#### Verify Access - Allowed Egress Access to Nginx

We can see that this is the case by switching over to our **access** pod in the namespace and attempting to access nginx.

```
wget -q --timeout=5 nginx -O -
```
It should return the HTML of the nginx welcome page.

```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>...
```

Next, try to retrieve the home page of google.com.

```
wget -q --timeout=5 google.com -O -
```

It should return:

```
wget: download timed out
```

Access to google.com times out because it can resolve DNS but has no egress access to anything other than pods with labels matching run: nginx in the advanced-policy-demo namespace.

## Summary

This has been a series of examples to help you understand network policy with Kubernetes and how it can be used to secure your cluster.  What next?  Run through these examples at home / work.  They can be found in similar form at Calico org.  Take it to the next level by deploying a service mesh with Istio.

> For the curious, explore the workload you have deployed.  Find the names of pods and services within the cluster.  Connect the names of pods to interfaces on the nodes that host them:  **kubectl get pods --all-namespaces -o wide** will help.  Run **ipconfig** and check the names of the interfaces against a comparison of the IPs for pods found in the results of an **ip routes** command.  You should be able to connect pods and services to routes and interfaces.  Good Luck!  Next, explore IPTables and find the chains that are associated with your interfaces.
