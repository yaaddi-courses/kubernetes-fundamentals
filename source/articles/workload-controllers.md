# Workload controllers: ReplicaSets, Deployments, StatefulSets and DaemonSets

**Covers:** ReplicaSet, Deployment, StatefulSet, DaemonSet, Namespace

## Why you do not create pods by hand {#overview}
Suppose you create a pod directly and its node fails. Who starts a new one? Nobody. A pod on its own has no one watching over it. That is the problem controllers solve. A controller is a loop that keeps comparing what you asked for with what exists, and acts whenever the two differ. Each controller type suits a different kind of workload, so choosing the right one is among the first decisions you make. Let's meet them one by one, starting with the way objects are kept apart.

## Namespace {#namespace}
Before the controllers, one question: what if two teams both want to call their application `web`? A namespace answers it. It is a separate space inside one cluster, and a name only has to be unique inside its own namespace, so each team can have its own `web`. You also attach access rules and resource limits to namespaces, which makes them the natural unit for a team or an environment. A new cluster starts with `default`, `kube-system` for its own parts, and a couple of others.

## ReplicaSet {#replicaset}
A ReplicaSet is the simplest controller. You give it a pod template and a number, such as "three copies". If one pod dies, it creates another. If someone starts an extra pod that matches, it deletes the surplus. You will almost never create a ReplicaSet yourself, because the controller above it is more useful, but you will see them constantly in listings.

## Deployment {#deployment}
The Deployment is the controller you will use day to day. It manages a ReplicaSet underneath and adds two things you want in real life. The first is gradual updates: when you change the image or the settings, it replaces pods a few at a time. The second is easy returns: you can go back to the previous version.

Here is how it works. When you update a Deployment, it creates a new ReplicaSet for the new version and slowly moves pods from the old one to the new one, so the application stays available. The old ReplicaSet is kept at zero pods, ready for a return. Use a Deployment for stateless applications, such as web servers and API programs, where any pod is as good as any other.

## StatefulSet {#statefulset}
Some programs are not interchangeable. A database member has a role and its own data. A StatefulSet treats its pods that way: each gets a stable name and its own storage that follows it. The pods are named in order, `db-0`, `db-1`, `db-2`, and they start and stop in that order. When `db-1` is replaced, it comes back as `db-1` with the same disk.

That makes it the right choice for databases and similar systems. It is more demanding to operate than a Deployment, so prefer a managed database offering when you can.

## DaemonSet {#daemonset}
A DaemonSet answers a different need: some things must exist on every machine. It starts exactly one copy of a pod on each node. When a node joins the cluster, it gets a copy, and when a node leaves, its copy goes with it. Log collectors, monitoring agents and network components are the usual examples.

## A worked example {#example}
Let's read a minimal Deployment and see how the pieces relate.

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

Here is what Kubernetes does with it:

1. The Deployment `shop-web` is created in the namespace `shop`.
2. It creates a ReplicaSet, which creates 3 pods from the template.
3. If you delete one pod, the ReplicaSet starts a replacement within seconds, so the count returns to 3.
4. If you change the image to `shop/web:1.5`, the Deployment creates a second ReplicaSet and shifts the pods across.

Let's count that update. With 3 replicas and the default rule, which allows 1 extra pod and 1 unavailable at a time, the sequence goes like this. Start 1 new pod, so there are 4 in all. Stop 1 old pod, so there are 3 in all, 1 of them new. Repeat until 3 are new and 0 are old. At every moment, at least 2 pods serve traffic. To go back, you use `kubectl rollout undo deployment/shop-web`.

Use this table to choose among the four:

| Need | Controller |
|---|---|
| Stateless application, gradual updates | Deployment (through a ReplicaSet) |
| Database, each member needs its own name and disk | StatefulSet |
| One agent on every node | DaemonSet |
| Keep N identical pods, without updates | ReplicaSet (rarely directly) |

A DaemonSet on a cluster with 10 nodes starts 10 pods, one per node. Add an eleventh node and it grows to 11 without any change to its definition.

## A common mistake {#mistake}
A common slip is using a Deployment for a database. Its pods get new names and lose their disks when replaced, so data is lost or confused. Use a StatefulSet, or better still a managed database.

Another slip is putting everything in the `default` namespace. It works at first, but it soon mixes teams and environments together, and separating them later is harder than starting with a few namespaces.

## Key points {#key-points}
- Controllers keep what exists equal to what you asked for, which is why you do not create pods by hand.
- A Deployment manages a ReplicaSet and adds gradual updates and easy returns to the previous version.
- A StatefulSet gives each pod a stable name and its own storage. A DaemonSet places one pod on every node.
- Namespaces separate teams and environments inside one cluster.
