# Securing an ICP Cluster with Policy

During this lab, you will explore how network policy can be used to secure workload in your IBM Cloud Private cluster.  

> If you manage an ICP cluster, you will likely want to provide additional security beyond the default behavior.  By design, Kubernetes has a principal that all pods within your cluster can communicate with all other pods within your cluster.  If you have time, after completing the lab you can explore the basic Linux networking settings configured on your hosts.  There will be some hints for this activity following the final section.

In the Basic Policy Example you will be guided through the following procedures:

* Setting your first basic network policy
* Configuring a frontend / backend policy example
* Controlling ingress and egress traffic using policy
* Applying application layer policy

## Basic Policy Example

### Configure Namespaces

This lab will deploy pods in a Kubernetes namespace **policy-demo**. Let’s create the Namespace object
```
kubectl create ns policy-demo
```

### Create Demo Pods

Use Kubernetes **Deployment** objects to easily create pods in the namespace.  Create some nginx pods in the policy-demo namespace.

```
kubectl run --namespace=policy-demo nginx --replicas=2 --image=nginx
```

Expose them through a service.

```
kubectl expose --namespace=policy-demo deployment nginx --port=80
```

Ensure the nginx service is accessible.

> Throughout these labs you will use this busybox pod **below** to probe your environment and determine the state of workload connectivity.  You will find it convenient to open a separate terminal window and leave this prompt running.

```
kubectl run --namespace=policy-demo access --rm -ti --image busybox /bin/sh
```

This should open up a shell session inside the access pod, as shown below.

```
Waiting for pod policy-demo/access-472357175-y0m47 to be running, status is Pending, pod ready: false

If you don't see a command prompt, try pressing enter.
/ #
```

From inside the access pod, attempt to reach the nginx service.

```
wget -q nginx -O -
```

You should see a response from nginx. Great! Our service is accessible. You can now exit the pod.

### Enable isolation

Let’s turn on isolation in our policy-demo namespace. Calico will then prevent connections to pods in this namespace.  

> This is one of the fundamental building blocks that you can use to secure workload within your clusters.  You will typically create isolated spaces within the cluster and purposefully return only desired connectivity.

Running the following command creates a NetworkPolicy which implements a default deny behavior for all pods in the policy-demo namespace.

```
kubectl create -f - <<EOF
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: default-deny
  namespace: policy-demo
spec:
  podSelector:
    matchLabels: {}
EOF
```

> Within our ICP cluster, Calico and Kubernetes have teamed up to create policy for controlling network behavior.  When initiating this policy from Kubernetes it is called **NetworkPolicy**.  You can also create this policy using the **calicoctl** interface.  In this case the syntax is different, and the policy is simply referred to as **policy**.  How does this network policy actually control connectivity within the cluster?  Simple, each Linux node within this cluster has **IPTables** configured.  This acts as a firewall each of your nodes.  If you are familiar with container and Kubernetes networks, you will know that each Pod simply gets an interface configured on the node it is hosted on.  Associated routes are configured (conveniently by Kubernetes and Calico) so that workloads can find each other (and your Ingress traffic can find its destination).  Of course **services** help all of this happen quite conveniently.  IPTables (or IPVS if you have configured that method when you deployed your cluster) provides some boundaries.

### Test Isolation

This will prevent all access to the nginx service. We can see the effect by trying to access the service again (or switch to the window where you have busybox already running).

```
kubectl run --namespace=policy-demo access --rm -ti --image busybox /bin/sh
```

This should open up a shell session inside the access pod, as shown below.

```
Waiting for pod policy-demo/access-472357175-y0m47 to be running, status is Pending, pod ready: false

If you don't see a command prompt, try pressing enter.

/ #
```

Now from within the busybox access pod execute the following command to test access to the nginx service.

```
wget -q --timeout=5 nginx -O -
```

The request should time out after 5 seconds.

```
wget: download timed out
/ #
```

By enabling isolation on the namespace, we’ve prevented access to the service.

> If you are familiar with IPTables you could inspect the configuration and find a Chain that denies this connectivity.

### Allow Access using a NetworkPolicy

Now, let’s enable access to the nginx service using a NetworkPolicy. This will allow incoming connections from our access pod, but not from anywhere else.

> This is a typical secure method of establishing only the connections you desire within the cluster.  This is how we build our secure point-to-point mesh.

Create a network policy access-nginx with the following contents:

```
kubectl create -f - <<EOF
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: access-nginx
  namespace: policy-demo
spec:
  podSelector:
    matchLabels:
      run: nginx
  ingress:
    - from:
      - podSelector:
          matchLabels:
            run: access
EOF
```

> The NetworkPolicy allows traffic from Pods with the label **run: access** to Pods with the label **run: nginx**. These are the labels automatically added to Pods started via kubectl run based on the name of the Deployment.

We should now be able to access the service from the access pod.

```
kubectl run --namespace=policy-demo access --rm -ti --image busybox /bin/sh
```

This should open up a shell session inside the access pod, as shown below.

```
Waiting for pod policy-demo/access-472357175-y0m47 to be running, status is Pending, pod ready: false

If you don't see a command prompt, try pressing enter.
/ #
```

Now from within the busybox access pod execute the following command to test access to the nginx service.

```
wget -q --timeout=5 nginx -O -
```

However, we still cannot access the service from a pod without the label **run: access**. We can verify this as follows.

```
kubectl run --namespace=policy-demo cant-access --rm -ti --image busybox /bin/sh
```

This should open up a shell session inside the **cant-access** pod, as shown below.

```
Waiting for pod policy-demo/cant-access-472357175-y0m47 to be running, status is Pending, pod ready: false

If you don't see a command prompt, try pressing enter.
/ #
```

Now from within the busybox cant-access pod execute the following command to test access to the nginx service.

```
wget -q --timeout=5 nginx -O -
```

The request should time out.

```
wget: download timed out
/ #
```

You can clean up the demo by deleting the demo namespace.

```
kubectl delete ns policy-demo
```

This was just a simple example of the Kubernetes NetworkPolicy API and how Calico can secure your Kubernetes cluster.

