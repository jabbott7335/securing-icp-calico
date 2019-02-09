# Running the Stars Example

This demo will give you a much more visual way of creating a zoned environment within your cluster.  After completing this section you will likely have a better idea of how you could recreate a typical tiered application network.  Creating traditional zoned environments might seem desirable, but try to think of your cluster network as point to point, only allowing the connections defined between only the workloads you wish to communicate.  

Create the frontend, backend, client, and management-ui apps.

```
kubectl create -f https://docs.projectcalico.org/v3.5/getting-started/kubernetes/tutorials/stars-policy/manifests/00-namespace.yaml
kubectl create -f https://docs.projectcalico.org/v3.5/getting-started/kubernetes/tutorials/stars-policy/manifests/01-management-ui.yaml
kubectl create -f https://docs.projectcalico.org/v3.5/getting-started/kubernetes/tutorials/stars-policy/manifests/02-backend.yaml
kubectl create -f https://docs.projectcalico.org/v3.5/getting-started/kubernetes/tutorials/stars-policy/manifests/03-frontend.yaml
kubectl create -f https://docs.projectcalico.org/v3.5/getting-started/kubernetes/tutorials/stars-policy/manifests/04-client.yaml
```

Wait for all the pods to enter Running state.

```
kubectl get pods --all-namespaces --watch
```

Note that it may take several minutes to download the necessary Docker images for this demo.

The management UI runs as a NodePort Service on Kubernetes, and shows the connectivity of the Services in this example.

You can view the UI by visiting `http://10.10.1.10:30002` in your browser.

Once all the pods are started, they should have full connectivity. You can see this by visiting the UI. Each service is represented by a single node in the graph.

* backend -> Node “B”
* frontend -> Node “F”
* client -> Node “C”

### Enable Isolation

Running following commands will prevent all access to the frontend, backend, and client Services.

```
kubectl create -n stars -f https://docs.projectcalico.org/v3.5/getting-started/kubernetes/tutorials/stars-policy/policies/default-deny.yaml
kubectl create -n client -f https://docs.projectcalico.org/v3.5/getting-started/kubernetes/tutorials/stars-policy/policies/default-deny.yaml
```

> Copy the URLs into your browser to follow the logic within the policy defenitions.  NetworkPolicy can be a little odd to read, consider less to be more in this syntax.

### Confirm Isolation

Refresh the management UI (it may take up to 10 seconds for changes to be reflected in the UI). Now that we’ve enabled isolation, the UI can no longer access the pods, and so they will no longer show up in the UI.

Next, allow the UI to access the Services using NetworkPolicy objects

Apply the following YAMLs to allow access from the management UI.

```
kubectl create -f https://docs.projectcalico.org/v3.5/getting-started/kubernetes/tutorials/stars-policy/policies/allow-ui.yaml
kubectl create -f https://docs.projectcalico.org/v3.5/getting-started/kubernetes/tutorials/stars-policy/policies/allow-ui-client.yaml
```

> Once again copy the URLs into your browser to follow the logic within the policy definitions. 

After a few seconds, refresh the UI - it should now show the Services, but they should not be able to access each other any more.

Create the backend-policy.yaml file to allow traffic from the frontend to the backend.

```
kubectl create -f https://docs.projectcalico.org/v3.5/getting-started/kubernetes/tutorials/stars-policy/policies/backend-policy.yaml
```
> View the policy defenition from within your browser. 

Refresh the UI. You should see the following:

* The frontend can now access the backend (on TCP port 6379 only).
* The backend cannot access the frontend at all.
* The client cannot access the frontend, nor can it access the backend.

Expose the frontend service to the client namespace.

```
kubectl create -f https://docs.projectcalico.org/v3.5/getting-started/kubernetes/tutorials/stars-policy/policies/frontend-policy.yaml
```

> View the policy definition from within your browser. 

The client can now access the frontend, but not the backend. Neither the frontend nor the backend can initiate connections to the client. The frontend can still access the backend.

