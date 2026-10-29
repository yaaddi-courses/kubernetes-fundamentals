# Inside the control plane and the node agents

**Covers:** API server, etcd, Kubelet, kubectl, Scheduler, Container runtime

## Who does what in a cluster {#overview}
When you ask Kubernetes to start something, a handful of small programs cooperate, each with one job. It helps to know the cast. When you read a log, meet an error or wonder why nothing is happening, you can then guess which member to look at first. Most of them live in the control plane, and two live on every node. Each talks to only a few others, and they all go through one central door. Let's meet them in the order you will use them.

## kubectl {#kubectl}
You will start with `kubectl`, the command-line tool for talking to a cluster. With it you ask questions, such as listing what is running, and you give orders, such as applying a configuration file. Every command becomes a request to the cluster's API server. Four you will use all the time: `kubectl get` lists objects, `kubectl describe` shows details and recent events, `kubectl apply -f file.yaml` creates or updates, and `kubectl logs` reads a program's output.

## API server {#api-server}
That request has to go somewhere, and it goes to the API server. It is the one door that every request passes through, whether it comes from you with `kubectl` or from a component inside Kubernetes. It checks who is asking, whether they are allowed and whether the request is valid. Then it records the change or returns the answer.

It is also the only component that reads and writes the cluster's database directly. All the others watch the API server for changes that concern them. That design gives you one place for security checks and one version of the truth.

## etcd {#etcd}
The database behind the API server is etcd. It stores every fact about the cluster's current state: which objects exist, their settings and where they live. It is a distributed key-value store, so a few copies, usually three or five, keep each other in step by voting, and the vote tolerates the loss of a minority. Only the API server talks to it.

Everything depends on etcd, which makes it the most important piece to back up. If its data is lost and there is no backup, the cluster forgets what it was supposed to host, even though the containers may still be alive on the nodes.

## Scheduler {#scheduler}
When a new workload appears with no node, the scheduler picks one. It reads what the workload asks for, such as processor and memory, and any placement rules. It filters out the nodes that cannot take the workload, scores the rest and picks the best. Notice that it only decides and writes the decision down. It does not start anything itself.

## Kubelet {#kubelet}
Someone has to carry out the decision, and that is the kubelet, an agent that sits on every node. It watches the API server for work assigned to its node and makes the work real by telling the container runtime to start, stop and monitor containers. It also reports the node's health and the state of each workload back to the API server. If the kubelet stops, the control plane sees the node as unhealthy.

## Container runtime {#container-runtime}
The container runtime is the low-level software that actually creates the containers on a node. It pulls images, starts and stops the processes and manages their isolation. Common choices are containerd and CRI-O, and Kubernetes talks to them through a standard interface called the Container Runtime Interface. Docker as a runtime was removed from Kubernetes in version 1.24. Images built with Docker still work, because the image format is a standard.

## A worked example {#example}
Let's follow one command through the system.

```bash
kubectl apply -f web.yaml
kubectl get pods
```

Step by step:

1. `kubectl` reads `web.yaml`, which asks for three copies of the web application, and sends it to the API server.
2. The API server checks your identity and the request, and stores the desired state in etcd.
3. The scheduler sees three new workloads with no node, picks a node for each and records its choice through the API server.
4. The kubelet on each chosen node sees the assignment and asks the container runtime to pull the image and start the container.
5. The kubelets report back that the containers are up. The second command shows them with the status `Running`.

Notice the pattern. The API server is the hub, and no component talks to another directly. Each one reads from it and writes to it.

The failure cases show why the work is split this way. If the scheduler is down for a few minutes, existing workloads keep going, but new ones wait without a node. If one kubelet dies, only its own node is affected. If etcd is lost, the whole cluster loses its memory, so it is replicated and backed up. A typical safe setup is three etcd copies, which survive the loss of one because the remaining two are still a majority, plus a nightly snapshot kept elsewhere.

## A common mistake {#mistake}
A common slip is looking for the problem in the wrong place. If a workload is stuck waiting with no node, look at the scheduler's events. If it has a node but no live container, look at the kubelet and the runtime. Keep this map in your head and you will save a lot of time.

Another slip is never backing up etcd. It feels like an advanced topic, until the day it is the only thing that matters.

## Key points {#key-points}
- `kubectl` sends requests to the API server, which is the only door into the cluster.
- etcd stores all of the cluster state. Back it up.
- The scheduler chooses a node, and the kubelet on that node starts the container through the container runtime.
- Components never talk to each other directly. They watch the API server.
