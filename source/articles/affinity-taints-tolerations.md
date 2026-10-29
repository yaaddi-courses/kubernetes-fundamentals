# Choosing nodes: affinity, taints and tolerations

**Covers:** Node affinity, Taint, Toleration

## Where a pod lands is not always up to the scheduler {#overview}
Left alone, the scheduler places a pod on any node with enough free resources. Most of the time that is what you want. Sometimes it is not. A pod that needs a fast disk should go to the nodes that have one, and a machine reserved for a special job should not fill up with ordinary pods. Kubernetes gives you two opposite tools for this: one attracts pods to nodes, and the other repels them. Let's look at both, and then at how they work together.

## Node affinity {#node-affinity}
Start with attraction. Node affinity is a rule you write on a pod, and it pulls the pod toward nodes that carry certain labels. A node might be labelled `disk=ssd` or `zone=north`, and the pod says, in effect, "place me on a node where `disk` is `ssd`".

The rule comes in two strengths. A required rule is strict: if no node matches, the pod waits. A preferred rule is a hint with a weight: the scheduler tries to match, but accepts another node if none does. Use required rules sparingly, because they make it easy to leave pods stranded.

## Taint {#taint}
Now the opposite direction. A taint is a mark on a node that repels pods. A tainted node accepts only pods that explicitly say they can live with the mark. The mark has three parts: a key, a value and an effect, written like `dedicated=gpu:NoSchedule`.

The effect decides how strong the repulsion is:
- `NoSchedule` stops new pods from being placed on the node.
- `PreferNoSchedule` makes the scheduler avoid the node where it can.
- `NoExecute` also removes pods already there that do not tolerate the mark.

## Toleration {#toleration}
A toleration is a declaration on a pod that lets it ignore a matching taint. Here is the part that trips people up: a toleration does not attract the pod to the node. It only lifts the ban. A toleration says "I am allowed", while affinity says "I want".

So to reserve nodes for one workload you need both. The taint keeps everyone else out, and affinity pulls the right pods in.

## A worked example {#example}
Let's work through a cluster of 5 nodes. Two of them, `gpu-1` and `gpu-2`, have graphics cards and are expensive. You want only the machine learning pods to use them.

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

Here is what each kind of pod gets:

| Pod | Taint on gpu nodes | Result |
|---|---|---|
| Ordinary web pod | Not tolerated | Cannot go to gpu-1 or gpu-2, so goes to one of the other 3 nodes |
| Training pod with a toleration only | Tolerated | May go to any of the 5 nodes, including a cheap one |
| Training pod with both | Tolerated, and affinity requires `hardware=gpu` | Goes only to gpu-1 or gpu-2 |

The third row is the one you want. With the toleration alone, the training pod could land on a plain node and fail for lack of a card. With affinity alone, ordinary pods could still crowd onto the gpu nodes. Only the pair reserves the nodes in both directions.

You will meet this pattern in real clusters. The control plane nodes carry a taint by default, so ordinary pods stay away from the cluster's own components.

## A common mistake {#mistake}
A common slip is adding a toleration and expecting the pod to go to the tainted node. The pod is only allowed there, and the scheduler may still choose another node. Add affinity when you want to attract it.

Another slip is a required affinity that no node satisfies, which leaves the pod waiting forever. When a pod will not start, `kubectl describe pod` shows the reason.

## Key points {#key-points}
- Node affinity attracts a pod to nodes by label. A taint repels pods from a node. A toleration cancels a taint for one pod.
- Required rules are strict, and preferred rules are hints.
- To dedicate nodes, taint the nodes and give the right pods both a toleration and affinity.
- `NoExecute` also evicts pods that are already there.
