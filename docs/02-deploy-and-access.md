# Milestone 2: First Workload and External Access

Covers LFS258 Labs 3.4 (Deploy a Simple Application) and 3.5 (Access from
Outside the Cluster), run against the self-hosted VMware cluster instead of the
browser lab. The clean path is below. The part worth keeping is the reachability
lesson at the end, because the lab step assumes a cloud node and quietly breaks
off-cloud.

## Cluster reference

| Node     | Role          | IP             |
| -------- | ------------- | -------------- |
| `cp`     | control plane | 192.168.239.50 |
| `worker` | worker        | 192.168.239.51 |

NodePort range is 30000 to 32767. This writeup uses 31720 as assigned in my run.
Yours will differ each time the service is recreated.

---

## Lab 3.4: Deploy a simple application

1. Create the deployment:
   `kubectl create deployment nginx --image=nginx`

2. Confirm the pod is scheduled and running, and see which node it landed on:
   `kubectl get pods -o wide`

3. Expose it inside the cluster as a ClusterIP service on port 80:
   `kubectl expose deployment nginx --port=80`

4. Look at the service and its backing endpoints. The endpoints object is the
   live list of pod IPs the service load-balances to:
   `kubectl get svc nginx`
   `kubectl get endpoints nginx`

5. Verify from inside the cluster. A throwaway pod can resolve the service by
   name and reach it on the ClusterIP:
   `kubectl run test --image=busybox --restart=Never -it --rm -- wget -qO- nginx`

Teaching point: the service keeps working as pods come and go because the
endpoints list is rebuilt from the pod selector. It is never pinned to a
specific pod IP.

---

## Lab 3.5: Access from outside the cluster

### Service discovery via environment variables (step 2)

Every pod gets service connection details injected as environment variables at
start time. Inspect them from inside a running pod:
`kubectl exec <nginx-pod> -- printenv | grep KUBERNETES`

Expected keys include `KUBERNETES_SERVICE_HOST=10.96.0.1` and
`KUBERNETES_SERVICE_PORT=443`. This is the environment-variable half of what the
lab intro calls "a DNS add-on or environment variables." In practice you rely on
the DNS add-on (CoreDNS) far more, because env vars are only injected for
services that already existed when the pod started.

### Swap ClusterIP for LoadBalancer

1. Delete the ClusterIP service and recreate it as LoadBalancer:
   `kubectl delete svc nginx`
   `kubectl expose deployment nginx --type=LoadBalancer`

2. Check it:
   `kubectl get svc nginx`

The EXTERNAL-IP shows as `<pending>` with a port mapping like `80:31720/TCP`.

Why pending: LoadBalancer type asks a cloud controller for an external IP. A
bare-metal, self-hosted cluster has no cloud controller to answer, so it stays
pending indefinitely. That is expected here, not a fault. The lab reaches the
service through its NodePort instead.

### Reach it from outside (the part the lab actually wants)

Open a browser on a machine that can route to the node subnet. My Windows host
has a VMnet8 adapter on 192.168.239.0/24, so it routes to the VMs directly. Hit
the node IP on the assigned NodePort:
`http://192.168.239.50:31720`

That returns the nginx welcome page. If the browser fails, confirm the NodePort
serves from the node itself first:
`curl 192.168.239.50:31720`

Port-forward alternative, no service type change needed, and useful on the exam:
`kubectl port-forward svc/nginx 8080:80`
then browse `http://localhost:8080` from wherever that command is running.

### Scaling drives availability (steps 7 and 8)

Scale to zero and the page stops serving once the pods terminate:
`kubectl scale deployment nginx --replicas=0`
`kubectl get pods`

Scale back up and it serves again:
`kubectl scale deployment nginx --replicas=2`
`kubectl get pods`

The service and NodePort never changed through any of this. Availability tracked
the pods, which is the endpoints lesson from 3.4 shown live.

### Cleanup

`kubectl delete deployment nginx`
`kubectl delete svc nginx`

Note: deleting the deployment does not remove the service or its endpoints.
Delete the service explicitly.

---

## Key takeaways

- LoadBalancer EXTERNAL-IP stays pending on a self-hosted cluster with no cloud
  controller. NodePort or port-forward is how you reach a service off-cloud.
- `curl ifconfig.io` returns the egress WAN IP, not the node IP. Off-cloud those
  are not the same address, which is why the lab step breaks on a NAT lab.
- Endpoints, not the service, track pod availability. Scaling to zero drops the
  page while the service object stays put.
- Deleting a deployment leaves its service and endpoints behind.

---

*Part of an ongoing certification journey toward the CKA.*
