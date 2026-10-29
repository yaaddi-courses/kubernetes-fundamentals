# Inside the control plane and the node agents

**Covers:** API server, etcd, Kubelet, kubectl, Scheduler, Container runtime

## Who does what in a cluster {#overview}
When you ask Kubernetes to run something, several small programs cooperate, each with one job. Knowing the cast helps when you read logs, see an error message or wonder why nothing is happening. Most of them live in the control plane, and two live on every node. Each one talks only to a few others, and they all go through one central door.

## kubectl {#kubectl}
`kubectl` is the command-line tool that you use to talk to a cluster. With it you ask questions, such as listing what is running, and you give orders, such as applying a configuration file. Every command becomes a request to the cluster's API server. A few you will use all the time: `kubectl get` to list objects, `kubectl describe` to see details and recent events, `kubectl apply -f file.yaml` to create or update and `kubectl logs` to read a program's output.

## API server {#api-server}
The API server is the one door that every request to a cluster passes through, whether it comes from you running `kubectl` or from a component inside Kubernetes. It checks who is asking, whether they are allowed and whether the request is valid, and then it records the change or returns the answer.

It is the only component that reads and writes the cluster's database directly. All the others watch the API server for changes that concern them. That design keeps one place for security checks and one version of the truth.

## etcd {#etcd}
etcd is the single database that stores every fact about a cluster's current state: what objects exist, their settings and where they run. It is a distributed key-value store, so a few copies, usually three or five, keep each other in step using a voting method that tolerates the loss of a minority. Only the API server talks to it.

Because everything depends on it, etcd is the most important piece to back up. If the data is lost and there is no backup, the cluster forgets what it was running, even though the containers may still be alive on the nodes.

## Scheduler {#scheduler}
The scheduler is the control-plane component that picks which node a new workload should run on. It looks at what the workload asks for, such as processor and memory, and at any placement rules, then filters out the nodes that cannot take it and scores the rest. The best one wins. It only makes the decision and writes it down. It does not start anything itself.

## Kubelet {#kubelet}
The kubelet is the agent that runs on every node. It watches the API server for work assigned to its node, and makes it real by telling the container runtime to start, stop and monitor containers. It also reports the node's health and the state of each workload back to the API server. If the kubelet stops, the control plane sees the node as unhealthy.

## Container runtime {#container-runtime}
The container runtime is the low-level software on a node that actually creates and runs containers: it pulls the images, starts and stops the processes and manages their isolation. Common choices are containerd and CRI-O. Kubernetes talks to them through a standard interface called the Container Runtime Interface. Docker as a runtime was removed from Kubernetes in version 1.24, but images built with Docker still run, since the image format is a standard.

## A worked example {#example}
Follow one command through the system.

```bash
kubectl apply -f web.yaml
kubectl get pods
```

What happens, step by step:

1. `kubectl` reads `web.yaml`, which says "run three copies of the web application", and sends it to the API server.
2. The API server checks your identity and the request, and stores the desired state in etcd.
3. The scheduler sees three new workloads with no node, picks a node for each and records its choice, through the API server.
4. The kubelet on each chosen node sees the assignment, and asks the container runtime to pull the image and start the container.
5. The kubelets report back that the containers are running. The second command shows them with the status `Running`.

Count the hops: one request passes the API server several times, once for each reader and writer. The API server is the hub, and no component talks to another directly.

The failure cases show the value of the split. If the scheduler is down for a few minutes, existing workloads keep running, but new ones wait without a node. If one kubelet dies, only its node is affected. If etcd is lost, the whole cluster loses its memory. That is why it is replicated and backed up. A typical safe setup is three etcd copies, which survive the loss of one (a majority of two is still alive), and a nightly snapshot kept elsewhere.

## A common mistake {#mistake}
Looking for a problem in the wrong place. If a workload is stuck waiting with no node, look at the scheduler's events. If it has a node but no running container, look at the kubelet and the runtime. Another mistake is not backing up etcd.

## Key points {#key-points}
- `kubectl` sends requests to the API server, which is the only door into the cluster.
- etcd stores all of the cluster state. Back it up.
- The scheduler chooses a node, and the kubelet on that node starts the container through the container runtime.
- Components never talk to each other directly. They watch the API server.
