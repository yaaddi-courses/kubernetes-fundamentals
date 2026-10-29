# Pods, sidecars and processor use

**Covers:** Pod, Sidecar container, CPU

## The unit that Kubernetes actually schedules {#overview}
You might expect Kubernetes to run containers directly. It does not. It runs pods, and a pod holds one or more containers. The extra layer seems odd at first, but it solves a real problem: some programs need to run side by side and share things, and must always be placed together. Understanding the pod is the key to everything that follows.

## Pod {#pod}
A pod is the smallest deployable unit in Kubernetes: one or more containers that always run together on the same node, sharing the same network and storage. Most pods hold a single container, and the pod is just its wrapper. When a pod has several, they share one network address, so they reach each other through `localhost`, and they can share files through common storage.

A pod is also temporary. It is created, runs and is eventually replaced by a new pod, with a different name and a different address, which is why you rarely create pods directly. You describe what you want in a higher-level object, and Kubernetes creates the pods for you.

## Sidecar container {#sidecar-container}
A sidecar container is a helper that shares a pod with the main application container, using the same network and storage. The name comes from the sidecar of a motorcycle: it rides along and adds something useful, without being part of the bike.

Typical sidecars:
- A log shipper that reads the application's log files from shared storage and sends them to a central store.
- A proxy that handles encryption or traffic rules for the application's network calls.
- A configuration reloader that watches for changes and signals the application.

The benefit is separation: the application stays simple, and the helper can be developed, updated and reused by many applications. They start and stop together, because they are in one pod.

## CPU {#cpu}
The CPU, or central processing unit, is the processor that executes a program's instructions. Every pod uses some of the node's CPU and memory, and a node has only so much. Kubernetes measures CPU in cores, and in thousandths of a core called millicores: `500m` means half a core. How much a pod needs and may use is something that you can declare, which the scheduler uses to decide where to put it.

## A worked example {#example}
A pod with an application and a log-shipping sidecar:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shop-web
spec:
  containers:
    - name: app
      image: shop/web:1.4
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
    - name: log-shipper
      image: shop/log-shipper:2.0
      volumeMounts:
        - name: logs
          mountPath: /logs
  volumes:
    - name: logs
      emptyDir: {}
```

Read it:

1. The pod has two containers, `app` and `log-shipper`.
2. Both mount the same temporary storage, named `logs`. The application writes its log files there, and the shipper reads them.
3. Both share the pod's network address and are always placed on the same node.
4. If the node fails, both are replaced together on another node.

Now count the resources. Suppose the app needs 500 millicores and the sidecar 100, so the pod uses 600 millicores, which is 0.6 of a core. A node with 4 cores, which is 4,000 millicores, can hold at most 6 such pods (4,000 divided by 600 is 6.67, rounded down to 6), before counting the node's own overhead.

The design choice is clear when you compare the alternatives. Without a sidecar, every application would need log-shipping code inside it, written again for each language. With a sidecar, you write the shipper once and attach it to any pod.

A rule of thumb: put containers in the same pod only if they must share fate and storage, as with a helper. If they can run and scale independently, such as a web server and its database, they belong in separate pods.

## A common mistake {#mistake}
Putting an application and its database in one pod because they are "related". They then scale and fail together, and the database cannot be replaced without restarting the application. Use separate pods. Another mistake is treating a pod as permanent, with an address that never changes.

## Key points {#key-points}
- A pod is the smallest unit that Kubernetes schedules, holding one or more containers that share a network and storage.
- Pods are temporary, and you normally create them through higher-level objects.
- A sidecar container is a helper in the same pod, such as a log shipper or a proxy.
- Pods use CPU, measured in cores and millicores, which affects where they can be placed.
