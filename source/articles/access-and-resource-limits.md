# Access control and limits

**Covers:** RBAC, RoleBindings, ResourceQuota, PodDisruptionBudget, PDB

## Sharing a cluster safely {#overview}
A cluster is rarely used by one person. Developers, tools and automated jobs all talk to the same API, and many teams run workloads side by side. Three questions follow. Who may do what? How much of the cluster may each team use? And how can maintenance be done without taking an application down? Kubernetes answers each with its own object.

## RBAC {#rbac}
RBAC, role-based access control, grants permissions to users and service accounts through roles, instead of giving everyone full access. A role is a list of allowed actions on kinds of objects, such as "read pods and their logs". Nothing is allowed unless a role says so, so the starting point is no access.

Two kinds of roles exist. A Role applies inside one namespace. A ClusterRole applies across the cluster, or to objects that belong to no namespace, such as nodes.

## RoleBindings {#rolebindings}
RoleBindings are the objects that connect a role to the people or accounts that get it. The role says what is allowed, and the binding says to whom. One role can have many bindings, one for each team or account. A RoleBinding gives the role within its own namespace. A ClusterRoleBinding gives it everywhere.

## ResourceQuota {#resourcequota}
A ResourceQuota is a cap on the total resources that a whole namespace may use, such as CPU, memory, storage and the number of objects. It stops one team from taking the whole cluster. When a new pod would push the namespace over its limit, the request is rejected at once, with a message that names the exceeded quota.

With a quota on CPU or memory, every pod in that namespace must state its own requests, or it is refused. A companion object, the limit range, supplies defaults for pods that do not.

## PodDisruptionBudget {#poddisruptionbudget}
A PodDisruptionBudget limits how many pods of an application may be down at once during a voluntary disruption, such as planned maintenance of a node. It names the pods by a selector and sets either `minAvailable` or `maxUnavailable`. When an administrator drains a node, the cluster respects the budget and moves pods away in a safe order, waiting where it must.

It does not protect against involuntary failures such as a crash of a machine, since nothing can ask permission for those.

## PDB {#pdb}
PDB is the usual short form of PodDisruptionBudget. In command output, `kubectl get pdb` lists the budgets with the number of allowed disruptions now. If that number is 0, a node drain will wait until a pod is healthy again.

## A worked example {#example}
A team called `shop` has its own namespace, and it needs three things.

First, access. A developer should read pods and logs, but not delete anything:

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

Test it: `kubectl auth can-i delete pods -n shop --as dana` prints `no`, and `get pods` prints `yes`.

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

If the pods already use 7.5 CPUs of requests, a new pod asking for 1 CPU would make 8.5, which is above 8, so it is rejected. The team must free capacity or ask for more.

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

With 5 pods and `minAvailable: 4`, only 1 pod may be evicted at a time. If an administrator drains two nodes holding 2 of the pods, the cluster evicts the first, waits until its replacement is ready, and only then the second. The application never has fewer than 4 pods serving.

A budget set too strictly causes a different problem. With `minAvailable: 5` on 5 pods, no pod can ever be evicted, and drains hang. Leave room for at least one disruption.

## A common mistake {#mistake}
Giving everyone the all-powerful `cluster-admin` role because it is quicker. One leaked token then controls the whole cluster. Start with narrow roles. Another mistake is a budget that allows zero disruptions, which blocks maintenance.

## Key points {#key-points}
- RBAC allows only what a role says. RoleBindings attach roles to users and accounts.
- A ResourceQuota caps a namespace's total use, and rejects requests over it.
- A PDB (PodDisruptionBudget) keeps a minimum number of pods running during planned maintenance.
- Test permissions with `kubectl auth can-i`.
