# What Kubernetes is and what it adds to Docker

**Covers:** Kubernetes, UIs, Docker, Orchestration, Cluster, Control plane, Control-plane, Node

## From one container to hundreds {#overview}
Running one container on your laptop is easy. Running hundreds of them for a real product, on many machines, with users who expect it never to go down, is a different problem altogether. Someone, or something, has to decide where each container goes, notice when one dies and add more when traffic grows. Kubernetes is the most widely used system for that job.

## Docker {#docker}
Docker is the tool that packages an application and everything it needs into a container, and runs that container on one machine. A container image is the same wherever it runs, which solves "it works on my computer". It does not decide which of fifty machines should run the next one, or what to do when a machine fails.

## Orchestration {#orchestration}
Orchestration is the ongoing work of deciding where each container should run, restarting it if it dies, and scaling it up or down as demand changes. Like the conductor of an orchestra, an orchestrator does not play any instrument itself. It coordinates many players.

The jobs include placing containers on machines with enough room, keeping the right number running, replacing failed ones, rolling out new versions gradually, connecting containers to each other and storing configuration. Doing these by hand works for three containers and fails for three hundred.

## Kubernetes {#kubernetes}
Kubernetes is an open-source system that runs and manages containers for you across a whole group of machines, deciding where each one runs and keeping it running. It began inside Google, which had run containers at huge scale with an internal system called Borg, and was released as open source in 2014. It is now maintained by the Cloud Native Computing Foundation. The name is Greek for helmsman, and it is often shortened to K8s.

You do not tell Kubernetes what steps to take. You tell it what you want, for example "keep five copies of this application running", and it works continuously to make reality match. If a machine fails, it starts replacements elsewhere on its own. This idea is called declarative configuration.

## Cluster {#cluster}
A cluster is the whole group of machines that Kubernetes manages together as one unit. From the outside, it looks like a single large computer on which you place your workloads. Inside, it is divided into a part that makes decisions and a set of machines that do the work.

## Control plane {#control-plane}
The control plane is the "brain" half of a cluster: the set of components that decide what should run and where, and watch to keep the cluster in its desired state. It does not run your applications. It receives your requests, stores the state, chooses placements and reacts to changes. In production, it runs on several machines, so that losing one does not stop the decisions.

## Node {#node}
A node is one of the worker machines in a cluster that actually runs your containers, as directed by the control plane. It may be a physical server or a virtual machine, and it runs a small agent that takes orders from the control plane. A cluster may have three nodes for a trial or thousands for a large company.

## UIs {#uis}
UIs are user interfaces: the screens and controls that people use to work with software. For Kubernetes, most people use a command-line tool, but graphical and terminal-based UIs exist too, such as the Kubernetes Dashboard, Lens and k9s. They show what is running and its health, which is easier to follow than raw text when a cluster is large.

## A worked example {#example}
A shop's product is made of three kinds of container: a web front end, an order handler and a payment connector. Compare two ways of running them.

| | With Docker only | With Kubernetes |
|---|---|---|
| Who decides where each one runs | A person | The system, using free capacity |
| A machine fails at 3 a.m. | The product is down until a person acts | The system restarts the lost containers on other nodes |
| Traffic doubles | A person starts more by hand | The system adds copies when told the rule |
| New version | A person stops and starts, with a short gap | A gradual replacement, one at a time |

Suppose the product needs 12 containers spread over 4 nodes. Each node runs 3. If one node fails, 3 containers vanish. Kubernetes notices within about a minute and starts 3 replacements on the 3 healthy nodes, which then carry 4 each. The product keeps working with a brief dip.

By hand, the same recovery takes however long it takes someone to wake up, log in, find which containers were on the broken machine and start them again, perhaps 30 minutes at night. If the product earns 600 dollars an hour, that is 300 dollars lost by one failure, which orchestration reduces to about a minute.

This does not mean that Kubernetes is always the right answer. It brings many concepts, and a small application on one or two servers may be better off without it.

## A common mistake {#mistake}
Adopting Kubernetes because it is fashionable, for a small system that Docker alone could run. The learning cost and running cost are real. Another mistake is thinking that it replaces Docker. It runs the same kind of containers, and Docker or a similar tool still builds the images.

## Key points {#key-points}
- Docker packages and runs containers on one machine. Orchestration decides where they run, restarts them and scales them.
- Kubernetes is the leading open-source orchestrator. You describe the state you want, and it keeps reality matching.
- A cluster has a control plane that decides and nodes that run the containers.
- Use it when you have enough containers and machines to need it.
