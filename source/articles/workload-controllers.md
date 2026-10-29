# Workload controllers: ReplicaSets, Deployments, StatefulSets and DaemonSets

**Covers:** ReplicaSet, Deployment, StatefulSet, DaemonSet, Namespace

## Why you do not create pods by hand {#overview}
A pod that you create directly is on its own. If it crashes, or its node fails, nobody replaces it. Controllers fix this. A controller is a loop that compares what you asked for with what exists and acts to close the gap. Each controller type suits a different kind of workload, and picking the right one is one of the first decisions you make.

## Namespace {#namespace}
A namespace is a logical space inside one cluster that lets different teams, environments or projects share it without their objects colliding. Names must be unique within a namespace, not across the cluster, so two teams can each have an object called `web` in separate namespaces. Namespaces are also the unit to which you attach access rules and resource limits. The cluster starts with `default`, `kube-system` for its own parts and a couple of others.

## ReplicaSet {#replicaset}
A ReplicaSet is a controller that keeps a fixed number of identical pods running at all times, automatically starting a replacement if one disappears. You give it a template for the pod and a number: "three copies". If one dies, it creates another. If someone starts an extra one that matches, it deletes it.

You almost never create a ReplicaSet yourself, since the controller above it is more useful, but you meet them constantly in listings.

## Deployment {#deployment}
A Deployment is the controller that most people use day to day. It manages a ReplicaSet underneath and adds two big things: gradual updates, which replace pods gradually when you change the image or settings, and easy returns, which go back to the previous version. When you update a Deployment, it creates a new ReplicaSet for the new version and slowly moves pods from the old one to the new one, so the application stays available. The old ReplicaSet is kept, scaled to zero, for going back.

Use it for stateless applications, such as web servers and API programs, where any pod is as good as any other.

## StatefulSet {#statefulset}
A StatefulSet is like a Deployment, but each pod gets a stable, unique identity and its own persistent storage that follows it. The pods are named in order, `db-0`, `db-1`, `db-2`, they start and stop in that order and a replaced `db-1` comes back with the same name and the same disk.

It is the right choice for databases and other systems where each member has a role and holds its own data. It is more demanding to run than a Deployment, so prefer a managed database offering when you can.

## DaemonSet {#daemonset}
A DaemonSet is a controller that runs exactly one copy of a pod on every node in the cluster. When a node joins, it gets a copy. When a node leaves, its copy goes too. It is the natural fit for things that must be everywhere: a log collector, a monitoring agent or a network component.

## A worked example {#example}
A minimal Deployment, and how the pieces relate:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shop-web
  namespace: shop
spec:
  replicas: 3
  selector:
    matchLabels:
      app: shop-web
  template:
    metadata:
      labels:
        app: shop-web
    spec:
      containers:
        - name: web
          image: shop/web:1.4
```

What Kubernetes does with it:

1. The Deployment `shop-web` is created in the namespace `shop`.
2. It creates a ReplicaSet, which creates 3 pods from the template.
3. If you delete one pod, the ReplicaSet starts a replacement within seconds, so the count returns to 3.
4. If you change the image to `shop/web:1.5`, the Deployment creates a second ReplicaSet and shifts the pods across.

Count the update. With 3 replicas and the default rule, which allows 1 extra pod and 1 unavailable at a time, the sequence is: start 1 new pod (4 in all), stop 1 old (3 in all, 1 new), and so on until 3 new and 0 old. At every moment, at least 2 pods serve traffic. Going back is `kubectl rollout undo deployment/shop-web`.

Choosing among the four:

| Need | Controller |
|---|---|
| Stateless application, gradual updates | Deployment (through a ReplicaSet) |
| Database, each member needs its own name and disk | StatefulSet |
| One agent on every node | DaemonSet |
| Keep N identical pods, without updates | ReplicaSet (rarely directly) |

A DaemonSet on a cluster with 10 nodes runs 10 pods, one per node, and grows to 11 if you add a node, with no change to the definition.

## A common mistake {#mistake}
Using a Deployment for a database. Pods get new names and lose their disks when replaced, so data is lost or confused. Use a StatefulSet, or better a managed database. Another mistake is putting everything in the `default` namespace, which mixes teams and environments.

## Key points {#key-points}
- Controllers keep what exists equal to what you asked for, so you do not create pods by hand.
- A Deployment manages a ReplicaSet and adds gradual updates and easy returns to the previous version.
- A StatefulSet gives each pod a stable name and its own storage. A DaemonSet runs one pod per node.
- Namespaces separate teams and environments inside one cluster.
