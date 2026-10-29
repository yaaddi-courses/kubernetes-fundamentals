# Choosing nodes: affinity, taints and tolerations

**Covers:** Node affinity, Taint, Toleration

## Where a pod lands is not always up to the scheduler {#overview}
Left alone, the scheduler places a pod on any node with enough free resources. Sometimes that is wrong. A pod that needs a fast disk should go to the nodes that have one. A machine reserved for a special job should not take ordinary pods. Kubernetes gives you two opposite tools: one that attracts pods to nodes, and one that repels them.

## Node affinity {#node-affinity}
Node affinity is a rule on a pod that attracts it toward nodes with certain labels. A node might carry `disk=ssd` or `zone=north`. The pod says: "place me on a node where `disk` is `ssd`".

There are two strengths. A required rule is strict: if no node matches, the pod stays waiting. A preferred rule is a hint with a weight: the scheduler tries to match, but accepts another node if none does. Use required rules sparingly, since they make pods easy to strand.

## Taint {#taint}
A taint is a mark on a node that repels pods. A tainted node accepts only pods that explicitly say they can live with the mark. It has three parts: a key, a value and an effect, written like `dedicated=gpu:NoSchedule`.

The effect says how strong the repulsion is. `NoSchedule` stops new pods from being placed there. `PreferNoSchedule` makes the scheduler avoid it where possible. `NoExecute` also removes pods already running there that do not tolerate it.

## Toleration {#toleration}
A toleration is a setting on a pod that lets it ignore a matching taint. It does not attract the pod to the node, it only removes the ban. This is the key difference from affinity: a toleration says "I am allowed", and affinity says "I want".

To reserve nodes for one workload, you therefore use both: the taint keeps others out, and affinity pulls the right pods in.

## A worked example {#example}
A cluster has 5 nodes. Two of them, `gpu-1` and `gpu-2`, have graphics cards and are expensive. You want only the machine learning pods to use them.

Step 1. Label and taint the two nodes:

```bash
kubectl label node gpu-1 hardware=gpu
kubectl label node gpu-2 hardware=gpu
kubectl taint node gpu-1 dedicated=gpu:NoSchedule
kubectl taint node gpu-2 dedicated=gpu:NoSchedule
```

Step 2. In the pod of the training job, add both settings:

```yaml
spec:
  tolerations:
    - key: dedicated
      operator: Equal
      value: gpu
      effect: NoSchedule
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: hardware
                operator: In
                values: [gpu]
```

What this gives:

| Pod | Taint on gpu nodes | Result |
|---|---|---|
| Ordinary web pod | Not tolerated | Cannot go to gpu-1 or gpu-2, so goes to one of the other 3 nodes |
| Training pod with a toleration only | Tolerated | May go to any of the 5 nodes, including a cheap one |
| Training pod with both | Tolerated, and affinity requires `hardware=gpu` | Goes only to gpu-1 or gpu-2 |

The third row is the one you want. With the toleration alone, the training pod could land on a plain node and fail for lack of a card. With affinity alone, ordinary pods could still crowd onto the gpu nodes. Only the pair reserves the nodes in both directions.

Many clusters already use this. The control plane nodes carry a taint by default, so that ordinary pods do not run beside the cluster's own components.

## A common mistake {#mistake}
Adding a toleration and expecting the pod to go to the tainted node. It only becomes allowed, and the scheduler may still choose another node. Use affinity to attract it. Another mistake is a required affinity that no node satisfies, leaving the pod waiting forever. Check with `kubectl describe pod`, which shows the reason.

## Key points {#key-points}
- Node affinity attracts a pod to nodes by label. A taint repels pods from a node. A toleration cancels a taint for one pod.
- Required rules are strict, preferred rules are hints.
- To dedicate nodes, taint the nodes and give the right pods both a toleration and affinity.
- `NoExecute` also evicts pods that are already running.
