# Pods, sidecars and processor use

**Covers:** Pod, Sidecar container, CPU

## The unit that Kubernetes actually schedules {#overview}
If you know containers, you might expect Kubernetes to place them directly. It does not. It places pods, and a pod holds one or more containers. The extra layer feels odd at first, but it solves a real problem: some programs must sit side by side and share things, so they must always land together. Once you understand the pod, much of what follows becomes easier.

## Pod {#pod}
A pod is the smallest unit Kubernetes deploys. Its containers always share one node, one network and their storage. Most pods hold a single container, and the pod is only its wrapper. When a pod holds several, they share one network address, so they reach each other through `localhost`, and they can exchange files through common storage.

Notice also that a pod is temporary. It is created, does its work and is eventually replaced by a new pod with a different name and a different address. That is why you rarely create pods directly. You describe what you want in a higher-level object, and Kubernetes creates the pods for you.

## Sidecar container {#sidecar-container}
Suppose your application needs a helper that must live next to it. A sidecar container is exactly that: a helper sharing the pod with the main container, with the same network and storage. The name comes from the sidecar of a motorcycle, which rides along and adds something useful without being part of the bike.

Typical sidecars are:
- A log shipper that reads the application's log files from shared storage and sends them to a central store.
- A proxy that handles encryption or traffic rules for the application's network calls.
- A configuration reloader that watches for changes and signals the application.

The benefit is separation. The application stays simple, and the helper can be developed, updated and reused by many applications. Because both are in one pod, they start and stop together.

## CPU {#cpu}
Every pod uses part of its node's CPU and memory, and a node has only so much. The CPU, the central processing unit, is the processor that carries out a program's instructions. Kubernetes measures it in cores and in thousandths of a core, called millicores, so `500m` means half a core. You can declare how much a pod needs and may use, and the scheduler uses those numbers to decide where the pod fits.

## A worked example {#example}
Let's build a pod with an application and a log-shipping sidecar:

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

Let's read it:

1. The pod has two containers, `app` and `log-shipper`.
2. Both mount the same temporary storage, named `logs`. The application writes its log files there, and the shipper reads them.
3. Both share the pod's network address and always land on the same node.
4. If the node fails, both are replaced together on another node.

Now count the resources. Suppose the app needs 500 millicores and the sidecar 100. Together the pod uses 600 millicores, which is 0.6 of a core. A node with 4 cores, which is 4,000 millicores, can hold at most 6 such pods: 4,000 divided by 600 is 6.67, rounded down to 6. That is before counting the node's own overhead.

Why was a sidecar the right design? Compare the alternative. Without one, every application would need log-shipping code inside it, written again for each language. With a sidecar you write the shipper once and attach it to any pod.

Here is a rule of thumb. Put containers in the same pod only if they must share fate and storage, like a main program and its helper. If they can live and scale independently, such as a web server and its database, they belong in separate pods.

## A common mistake {#mistake}
A common slip is putting an application and its database in one pod because they are "related". They then scale and fail together, and you cannot replace the database without restarting the application. Use separate pods.

Another slip is treating a pod as permanent, with an address that never changes. It will change, so reach it through a stable name instead.

## Key points {#key-points}
- A pod is the smallest unit Kubernetes schedules, holding one or more containers that share a network and storage.
- Pods are temporary, and you normally create them through higher-level objects.
- A sidecar container is a helper in the same pod, such as a log shipper or a proxy.
- Pods use CPU, measured in cores and millicores, which affects where they can be placed.
