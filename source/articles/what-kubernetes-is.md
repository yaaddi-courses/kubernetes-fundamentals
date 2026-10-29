# What Kubernetes is and what it adds to Docker

**Covers:** Kubernetes, UIs, Docker, Orchestration, Cluster, Control plane, Control-plane, Node

## From one container to hundreds {#overview}
You may already have started a container on your laptop, and when it breaks you notice at once. Now picture hundreds of containers, spread over many machines, for a product whose users expect it never to go down. Someone, or something, has to decide where each container goes, notice when one dies and add more when traffic grows. Kubernetes is the most widely used system for that work, and this article shows what it adds to Docker.

## Docker {#docker}
Let's begin with what Docker gives you. It packages an application and everything the application needs into a container image, and it starts that image as a container on one machine. The image is identical wherever you start it, which ends the old complaint "it works on my computer".

Now ask what Docker does not do. It does not decide which of fifty machines should take the next container, and it does not notice a machine failing and move its work elsewhere. Those questions appear only with many containers.

## Orchestration {#orchestration}
The missing piece is orchestration. The word comes from the conductor of an orchestra, who plays no instrument but keeps many players together. An orchestrator works the same way: it never does your application's work, it coordinates the containers that do.

The job has several parts. It places containers on machines that have room. It keeps the right number alive. It replaces failed ones. It rolls out a new version a little at a time, it connects containers to each other and it stores their configuration. By hand, that works for three containers and fails for three hundred.

## Kubernetes {#kubernetes}
Kubernetes is the open-source orchestrator most teams choose. It started at Google, which had operated containers at huge scale with an internal system called Borg, and it was released to the public in 2014. Today the Cloud Native Computing Foundation maintains it. The name is Greek for helmsman, often written K8s.

Here is the idea that makes it different. You do not give Kubernetes a list of steps. You tell it the result you want, for example "keep five copies of this application alive", and it works continuously to make reality match your request. If a machine fails, it starts replacements elsewhere without being asked again. We call this style declarative configuration: you declare the goal, and the system finds the steps.

## Cluster {#cluster}
When you use Kubernetes, you do not talk to one machine. You talk to a cluster, which is the whole group of machines it manages together. From the outside, it looks like one large computer. Inside, one part makes decisions and another does the work.

## Control plane {#control-plane}
Think of the control plane as the brain of the cluster. It receives your requests, remembers the state you asked for, chooses where each container should go and reacts when something changes. It does not host your applications itself. In production it is spread over several machines, so losing one does not stop the decisions.

## Node {#node}
The other part is the nodes. A node is a worker machine, either a physical server or a virtual machine, and it is where your containers actually live. Each node has a small agent that takes orders from the control plane. A trial cluster might have three nodes, and a large company may have thousands.

## UIs {#uis}
You will mostly work with Kubernetes from a command line, but you are not limited to it. Graphical and terminal-based user interfaces exist too, such as the Kubernetes Dashboard, Lens and k9s. They draw what is happening and how healthy it is, which is easier to read than raw text in a large cluster.

## A worked example {#example}
Let's compare two ways to operate a shop whose product has three kinds of container: a web front end, an order handler and a payment connector.

| | With Docker only | With Kubernetes |
|---|---|---|
| Who decides where each one goes | A person | The system, using free capacity |
| A machine fails at 3 a.m. | The product is down until a person acts | The system restarts the lost containers on other nodes |
| Traffic doubles | A person starts more by hand | The system adds copies when told the rule |
| New version | A person stops and starts, with a short gap | A gradual replacement, one at a time |

Now put numbers on it. Say the product needs 12 containers spread over 4 nodes, so each node holds 3. One node fails, and 3 containers vanish. Kubernetes notices within about a minute and starts 3 replacements on the 3 healthy nodes. Those nodes now carry 4 each, because 3 + 1 = 4, and 3 nodes with 4 containers make the full 12 again. The product dips briefly and keeps working.

By hand, someone has to wake up, log in, find the lost containers and start them again, perhaps 30 minutes at night. If the product earns 600 dollars an hour, half an hour costs 300 dollars for a single failure. Orchestration shrinks that gap to about a minute.

## A common mistake {#mistake}
A common slip is adopting Kubernetes because it is fashionable, for a small system that Docker alone could handle. Both the learning cost and the running cost are real.

A second slip is thinking Kubernetes replaces Docker. It manages the same kind of containers, and Docker or a similar tool still builds the images.

## Key points {#key-points}
- Docker packages an application and starts it on one machine. Orchestration decides where containers go, replaces them and scales them.
- Kubernetes is the leading open-source orchestrator. You describe the state you want, and it keeps reality matching.
- A cluster has a control plane that decides and nodes that host the containers.
- Choose it when you have enough containers and machines to need it.
