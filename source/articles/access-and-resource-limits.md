# Access control and limits

**Covers:** RBAC, RoleBindings, ResourceQuota, PodDisruptionBudget, PDB

## Sharing a cluster safely {#overview}
A cluster is rarely yours alone. Developers, tools and automated jobs all talk to the same API, and several teams place their workloads side by side. As soon as that happens, three questions come up. Who may do what? How much of the cluster may each team use? And how can someone maintain a machine without taking an application down? Kubernetes answers each question with its own object, and we will take them in that order.

## RBAC {#rbac}
Let's start with the first question. RBAC stands for role-based access control. Instead of giving everyone full access, you describe permissions in roles and then hand the roles to users and service accounts. A role is a list of allowed actions on kinds of objects, such as "read pods and their logs". Nothing is allowed unless some role says so, so the starting point is no access at all.

There are two kinds of roles, and the difference is about reach. A Role applies inside one namespace. A ClusterRole applies across the cluster, or to objects that belong to no namespace, such as nodes.

## RoleBindings {#rolebindings}
A role by itself does nothing, because nobody holds it yet. RoleBindings fix that. They connect a role to the people or accounts that should get it: the role says what is allowed, and the binding says to whom. One role can have many bindings, perhaps one for each team. A RoleBinding grants the role inside its own namespace, while a ClusterRoleBinding grants it everywhere.

## ResourceQuota {#resourcequota}
Now the second question. A ResourceQuota is a cap on the total resources a whole namespace may use: CPU, memory, storage and the number of objects. It stops one team from taking the entire cluster. When a new pod would push the namespace over its limit, the request is rejected at once, and the message names the quota that was exceeded.

One consequence surprises people. With a quota on CPU or memory, every pod in that namespace must state its own requests, or it is refused. A companion object, the limit range, supplies defaults for pods that do not.

## PodDisruptionBudget {#poddisruptionbudget}
The third question is about maintenance. Say an administrator wants to empty a node to update it. A PodDisruptionBudget limits how many pods of an application may be down at once during that kind of planned, voluntary disruption. It picks the pods with a selector and sets either `minAvailable` or `maxUnavailable`. When the administrator drains the node, the cluster respects the budget and moves pods away in a safe order, waiting where it has to.

Notice what it cannot do. It gives no protection against involuntary failures, such as a machine crashing, because nothing can ask permission for those.

## PDB {#pdb}
PDB is the everyday short form of PodDisruptionBudget. In command output, `kubectl get pdb` lists the budgets together with the number of disruptions allowed right now. If that number is 0, a node drain waits until a pod is healthy again.

## A worked example {#example}
Let's give a team called `shop` its own namespace and cover all three needs.

First, access. A developer should be able to read pods and logs, but not delete anything:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: shop
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: shop
  name: dev-reads-pods
subjects:
  - kind: User
    name: dana
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Test it. `kubectl auth can-i delete pods -n shop --as dana` prints `no`, and the same question about `get pods` prints `yes`.

Second, a quota:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  namespace: shop
  name: shop-quota
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    pods: "40"
```

Suppose the pods already use 7.5 CPUs of requests. A new pod asking for 1 CPU would bring the total to 8.5, which is above 8, so Kubernetes rejects it. The team must free capacity or ask for more.

Third, a budget for the web Deployment of 5 pods:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  namespace: shop
  name: shop-web
spec:
  minAvailable: 4
  selector:
    matchLabels:
      app: shop-web
```

With 5 pods and `minAvailable: 4`, only 1 pod may be evicted at a time. If an administrator drains two nodes that hold 2 of the pods, the cluster evicts the first, waits until its replacement is ready and only then evicts the second. The application never has fewer than 4 pods serving.

A budget can also be too strict. With `minAvailable: 5` on 5 pods, no pod can ever be evicted, and drains hang. Always leave room for at least one disruption.

## A common mistake {#mistake}
A common slip is giving everyone the all-powerful `cluster-admin` role because it is quicker. Then one leaked token controls the whole cluster. Start with narrow roles and widen them only when someone has a real need.

A second slip is a budget that allows zero disruptions. It looks safe, but it blocks every planned maintenance.

## Key points {#key-points}
- RBAC allows only what a role says, and RoleBindings attach roles to users and accounts.
- A ResourceQuota caps the total use of a namespace and rejects requests over it.
- A PDB (PodDisruptionBudget) keeps a minimum number of pods available during planned maintenance.
- Test permissions with `kubectl auth can-i`.
