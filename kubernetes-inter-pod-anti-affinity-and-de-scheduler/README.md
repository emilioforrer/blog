# Kubernetes Inter-pod anti-affinity and de-schedule


## Introduction

Before talking about affinity and anti-affinity inside a Kubernetes cluster, let's first understand what  Kubernetes is. Kubernetes is a platform to manage and orchestrate workloads and services based on containers that offer a lot of features such as auto-scaling (vertical and horizontal), container replicas, secrets management, etc.

Kubernetes also offers 2 important scheduling features that can be configured, to place pods inside the nodes. Those features are:

* **Node affinity**: This is similar to `nodeSelector` with the difference that the language is more expressive and you can create rules that are not **hard requirements** but rather a **soft/preferred** rule, meaning that the scheduler will still be able to schedule your pod, even if the rules can not be met.
* **Inter-pod affinity and anti-affinity**: Allow you to define rules that constrain which nodes your pod is eligible to be scheduled based on labels on pods that are already running on the node rather than based on labels on nodes. 

The focus of this post will be on **Inter-pod anti-affinity**:

* How to deploy an application with a rule that specifies to prefer scheduling a pod into another node if that node already contains a pod with the same labels as the pod to be scheduled (like in the case of multiple replicas of the same app)
* Fix a real-world edge case that can make your pods get stuck on the same node, even if you had specified a pod anti-affinity rule.

So let's get started!.


## Creating a local multi-node cluster

To create a local multi-node cluster in our machine, we will be using [Kind](https://kind.sigs.k8s.io/), so let's go ahead and follow the [installation guide](https://kind.sigs.k8s.io/docs/user/quick-start/#installation).

Once installed, let's create a configuration file (`kind-config.yaml`), specifying a cluster with 1 control-plane and 3 worker nodes:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
# One control plane node and three "workers".
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
```

Now let's create a cluster by running the following command:

```bash
kind create cluster --name k8s-playground --config kind-config.yaml
```

**Note**: It may take a few minutes, depending on your computer's resources

Let's check if the cluster was created and has the right nodes, to do this, run the following command:

```bash
kubectl get nodes
```

Should see an output similar to this one 

```bash
NAME                           STATUS   ROLES                  AGE   VERSION
k8s-playground-control-plane   Ready    control-plane,master   51s   v1.21.1
k8s-playground-worker          Ready    <none>                 25s   v1.21.1
k8s-playground-worker2         Ready    <none>                 25s   v1.21.1
k8s-playground-worker3         Ready    <none>                 25s   v1.21.1
```

Now, create a new deployment file (`deployment.yaml`) with the name `demo-app`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: demo-app
  name: demo-app
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels: # These are the the Pod labels
        app: demo-app
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions: # The key and value of the label that you will match against
                - key: app
                  operator: In
                  values:
                  - demo-app # In this example we are matching against the same labels as the pod label
              topologyKey: kubernetes.io/hostname
      containers:
      - image: nginxdemos/hello
        imagePullPolicy: Always
        name: hello
        resources: {}
```

Let's apply the deployment

```bash
kubectl apply -f deployment.yaml
```

Run `kubectl get pods -o wide` to see the running pods

```bash
NAME                       READY   STATUS    RESTARTS   AGE   IP           NODE                     NOMINATED NODE   READINESS GATES
demo-app-99d479bc9-w6f6p   1/1     Running   0          4s    10.244.1.7   k8s-playground-worker3   <none>           <none>
demo-app-99d479bc9-xhfj8   1/1     Running   0          4s    10.244.3.6   k8s-playground-worker    <none>           <none>
demo-app-99d479bc9-xwsfk   1/1     Running   0          4s    10.244.2.8   k8s-playground-worker2   <none>           <none>
```

As you can see, Kubernetes will prefer to place the pods on nodes that do not have an instance of the app running.

What happens if we have more replicas than the number of nodes? Well, let's see. Run the following command to scale the deployment to 5 replicas:

```bash
kubectl scale deployment demo-app --replicas 5
```

Now run `kubectl get pods -o wide`, the output should be similiar to this one

```bash
NAME                       READY   STATUS    RESTARTS   AGE    IP           NODE                     NOMINATED NODE   READINESS GATES
demo-app-99d479bc9-blmdm   1/1     Running   0          14s    10.244.1.8   k8s-playground-worker3   <none>           <none>
demo-app-99d479bc9-td9rk   1/1     Running   0          14s    10.244.3.7   k8s-playground-worker    <none>           <none>
demo-app-99d479bc9-w6f6p   1/1     Running   0          5m5s   10.244.1.7   k8s-playground-worker3   <none>           <none>
demo-app-99d479bc9-xhfj8   1/1     Running   0          5m5s   10.244.3.6   k8s-playground-worker    <none>           <none>
demo-app-99d479bc9-xwsfk   1/1     Running   0          5m5s   10.244.2.8   k8s-playground-worker2   <none>           <none>
```

As you can see, since we are using `preferredDuringSchedulingIgnoredDuringExecution`, Kubernetes **"preferred"** to place the other 2 replicas on nodes that already had pods of the same app running since there wasn't another node to schedule to.

## Taking down nodes

What happens if a node goes down? Well, let's find out.

Let's drain the node `k8s-playground-worker3` to simulate that node went down.

```bash
kubectl drain k8s-playground-worker3 --ignore-daemonsets
```

If we run `kubectl get pods -o wide`, we can see that all pods got rescheduled on nodes `k8s-playground-worker` and `k8s-playground-worker2`, since `k8s-playground-worker3` went down.

```bash
NAME                       READY   STATUS    RESTARTS   AGE     IP           NODE                     NOMINATED NODE   READINESS GATES
demo-app-99d479bc9-2ztpg   1/1     Running   0          2m11s   10.244.2.9   k8s-playground-worker2   <none>           <none>
demo-app-99d479bc9-c6pfn   1/1     Running   0          2m11s   10.244.3.9   k8s-playground-worker    <none>           <none>
demo-app-99d479bc9-td9rk   1/1     Running   0          20m     10.244.3.7   k8s-playground-worker    <none>           <none>
demo-app-99d479bc9-xhfj8   1/1     Running   0          25m     10.244.3.6   k8s-playground-worker    <none>           <none>
demo-app-99d479bc9-xwsfk   1/1     Running   0          25m     10.244.2.8   k8s-playground-worker2   <none>           <none>
```

Now, let's drain the node `k8s-playground-worker2`.

```bash
kubectl drain k8s-playground-worker2 --ignore-daemonsets
```

If we run `kubectl get pods -o wide`, we can see (as expected), that all the pods are running only in the node `k8s-playground-worker`, since  there is no other node in the cluster.

```bash
NAME                       READY   STATUS    RESTARTS   AGE    IP            NODE                    NOMINATED NODE   READINESS GATES
demo-app-99d479bc9-87pwh   1/1     Running   0          47s    10.244.3.11   k8s-playground-worker   <none>           <none>
demo-app-99d479bc9-c6pfn   1/1     Running   0          7m5s   10.244.3.9    k8s-playground-worker   <none>           <none>
demo-app-99d479bc9-kvwq7   1/1     Running   0          47s    10.244.3.10   k8s-playground-worker   <none>           <none>
demo-app-99d479bc9-td9rk   1/1     Running   0          25m    10.244.3.7    k8s-playground-worker   <none>           <none>
demo-app-99d479bc9-xhfj8   1/1     Running   0          30m    10.244.3.6    k8s-playground-worker   <none>           <none>
```

## Restoring down nodes

So, what happens if a node goes back online?

Let's see, run the following commands to restore the nodes 

```bash
kubectl uncordon k8s-playground-worker2
kubectl uncordon k8s-playground-worker3
```

Wait a moment then run the following command to see if all nodes are back online.


```bash
kubectl get nodes -o wide
```

Should print an output similar to this one

```bash
NAME                           STATUS   ROLES                  AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE       KERNEL-VERSION                      CONTAINER-RUNTIME
k8s-playground-control-plane   Ready    control-plane,master   64m   v1.21.1   172.29.0.5    <none>        Ubuntu 21.04   5.10.60.1-microsoft-standard-WSL2   containerd://1.5.2
k8s-playground-worker          Ready    <none>                 64m   v1.21.1   172.29.0.2    <none>        Ubuntu 21.04   5.10.60.1-microsoft-standard-WSL2   containerd://1.5.2
k8s-playground-worker2         Ready    <none>                 64m   v1.21.1   172.29.0.4    <none>        Ubuntu 21.04   5.10.60.1-microsoft-standard-WSL2   containerd://1.5.2
k8s-playground-worker3         Ready    <none>                 64m   v1.21.1   172.29.0.3    <none>        Ubuntu 21.04   5.10.60.1-microsoft-standard-WSL2   containerd://1.5.2
```

We can see that all the nodes are back online!

## What happened to the pods?

Run `kubectl get pods -o wide`, to list the pods.

```bash
NAME                       READY   STATUS    RESTARTS   AGE   IP            NODE                    NOMINATED NODE   READINESS GATES
demo-app-99d479bc9-87pwh   1/1     Running   0          16m   10.244.3.11   k8s-playground-worker   <none>           <none>
demo-app-99d479bc9-c6pfn   1/1     Running   0          22m   10.244.3.9    k8s-playground-worker   <none>           <none>
demo-app-99d479bc9-kvwq7   1/1     Running   0          16m   10.244.3.10   k8s-playground-worker   <none>           <none>
demo-app-99d479bc9-td9rk   1/1     Running   0          41m   10.244.3.7    k8s-playground-worker   <none>           <none>
demo-app-99d479bc9-xhfj8   1/1     Running   0          46m   10.244.3.6    k8s-playground-worker   <none>           <none>
```

**What?!**, all pods are still running on node `k8s-playground-worker`, even if all the other nodes are back online!. 

**What does this mean?**

* If node `k8s-playground-worker` goes down, we will have downtime in our application during the re-scheduling to the other nodes. Since all the pods are on the same node
* We have lost high availability (HA) in our cluster for that app, even when having multiple nodes up and running.

## The issue

What happened was that the inter-pod anti-affinity mechanism is **only relevant during scheduling**. Once a pod is running, the rules cannot be re-applied. To apply the rules again, you will need to recreate the pod.

## The solution

To fix this, we need to somehow watch for node changes and reapply the rules and de-schedule the pods to distribute the workload accordingly to the rules specification.

Luckily there is already a tool that does that. It is called [Descheduler](https://github.com/kubernetes-sigs/descheduler).

Let's install the [Helm Chart](https://github.com/kubernetes-sigs/descheduler/blob/master/charts/descheduler/README.md), by running the following command.

```bash
helm repo add descheduler https://kubernetes-sigs.github.io/descheduler/
helm install descheduler --namespace kube-system descheduler/descheduler
```

And that's all, only need to wait a few minutes to take effect. Run `kubectl get pods -o wide` to watch the changes in the pods.

```bash
NAME                       READY   STATUS    RESTARTS   AGE   IP            NODE                     NOMINATED NODE   READINESS GATES
demo-app-99d479bc9-9vxbr   1/1     Running   0          45s   10.244.2.12   k8s-playground-worker2   <none>           <none>
demo-app-99d479bc9-s95f8   1/1     Running   0          45s   10.244.1.11   k8s-playground-worker3   <none>           <none>
demo-app-99d479bc9-td9rk   1/1     Running   0          91m   10.244.3.7    k8s-playground-worker    <none>           <none>
demo-app-99d479bc9-xgjgz   1/1     Running   0          45s   10.244.2.11   k8s-playground-worker2   <none>           <none>
demo-app-99d479bc9-xhfj8   1/1     Running   0          96m   10.244.3.6    k8s-playground-worker    <none>           <none>
```

As you can see, the anti-affinity rules got reapplied, and the pods are re-scheduled on different nodes again. High availability for your application has been restored!.

**Note**: [Descheduler](https://github.com/kubernetes-sigs/descheduler) has a lot more options, but that's a story for another post.


## Summary

We have learned how to create an application and applied anti-affinity rules to spread the pods across the nodes and avoid all replicas of the app to schedule on the same node, achieving more high availability. We also learned that some edge cases make our apps lose high availability and introduce downtime. And to fix those kinds of situations, there are tools like [Descheduler](https://github.com/kubernetes-sigs/descheduler), that can help us overcome those issues.

If you want to learn more about how to assign pods to nodes, you can check the official documentation [here](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/).

Thanks for your time reading this article. See you on the next one!