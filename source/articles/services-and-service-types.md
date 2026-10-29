# Services: one stable address for changing pods

**Covers:** Service, IP, IPs, ClusterIP, NodePort, LoadBalancer

## The problem with addresses that keep changing {#overview}
A pod gets a network address when it starts, and loses it when it is replaced. Replacements, restarts and scaling happen all the time, so the set of pods that serve an application, and their addresses, is never the same for long. A program that needs to reach "the web application" cannot keep a list of addresses. Kubernetes solves this with an object that sits in front of the pods and gives them one fixed address.

## IP {#ip}
An IP address, short for Internet Protocol address, is the number that identifies a device or a pod on a network, such as `10.244.1.17`. Every pod in a cluster gets its own address from the cluster's internal range, and any pod can reach any other by it. But the address belongs to that pod only for as long as the pod lives. A new pod gets a new one.

## IPs {#ips}
IPs, the plural, are those addresses in general. A Service has its own IP, called the cluster IP, which does not change for the life of the Service, even while the pods behind it come and go. A listing of the pods behind a Service shows several different IPs, one per pod, and those are the ones that change.

## Service {#service}
A Service is an object that gives a changing set of pods one stable network address, so that other things can reach them reliably, even as individual pods are replaced. It picks its pods by a matching key-value pair, and keeps the list up to date by itself. Requests sent to the Service are spread across the healthy pods behind it.

The matching is done with a query: "send traffic to every pod whose `app` pair is `shop-web`". When such a pod starts and passes its readiness check, it joins. When it is deleted or fails, it leaves.

## ClusterIP {#clusterip}
ClusterIP is the default type of Service. It gives the pods a stable address that can be reached only from inside the cluster, not from the internet. It is what you use for internal connections, such as a web application calling a database or another application. Because it is internal, it is the safest choice, and most Services in a cluster are of this type.

## NodePort {#nodeport}
NodePort is a Service type that opens the same port on every node in the cluster, so that the application can be reached from outside by using any node's address and that port. The port is in a fixed range, 30000 to 32767 by default. A request to `node-1:31080` and to `node-2:31080` both reach the Service.

It is simple and needs no cloud features, so it is common in test clusters. But the high port numbers are awkward, and you must manage how clients find a healthy node, which is why production setups rarely rely on it alone.

## LoadBalancer {#loadbalancer}
LoadBalancer is a Service type that asks the cloud provider to create a real external load balancer with an internet-facing address, and to send its traffic to the Service. It is the straightforward way to expose an application to the internet on a cloud platform. It builds on the other two: the provider's balancer sends traffic to the nodes, and the Service spreads it across the pods.

The cost is that each LoadBalancer Service usually creates its own cloud load balancer, which is billed by the hour. Fifty of them would be expensive, which is one reason for sharing a single entry point through an ingress.

## A worked example {#example}
A Service for the web pods:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: shop-web
spec:
  type: ClusterIP
  selector:
    app: shop-web
  ports:
    - port: 80
      targetPort: 8000
```

How it works:

1. The Service gets a stable cluster IP, for example `10.96.14.20`, which stays the same for as long as the Service exists.
2. It picks all pods with the pair `app=shop-web`, say three of them with the changing IPs `10.244.1.17`, `10.244.2.9` and `10.244.3.4`.
3. A request to `10.96.14.20:80` is sent to one of those pods, on its port 8000.
4. If a pod is replaced, and its IP changes, the Service updates its list. Callers notice nothing.

The same Service with a different type changes only who can reach it:

| Type | Reachable from | Typical use |
|---|---|---|
| `ClusterIP` | Inside the cluster only | Internal connections |
| `NodePort` | Outside, at any node's IP and a port from 30000 to 32767 | Tests, simple set-ups |
| `LoadBalancer` | Outside, at one cloud address | Public applications |

Count the balancers: with 12 public applications, 12 LoadBalancer Services mean 12 cloud load balancers. At, say, 20 dollars a month each, that is 240 dollars. A single shared entry point costs 20. That is why many teams use one load balancer with routing rules behind it.

Test it from inside the cluster: `kubectl run tmp --rm -it --image=busybox -- wget -qO- shop-web`. The name works because of the cluster's own name service, described in the next article.

## A common mistake {#mistake}
Using a pod's IP in a configuration file. It stops working at the next restart. Always connect to a Service. Another mistake is a query that does not match the pods' pairs, so that the Service has no targets and every request fails. Check with `kubectl get endpoints shop-web`.

## Key points {#key-points}
- Pod IPs change. A Service has one stable IP and spreads requests across the matching pods.
- `ClusterIP` is internal only. `NodePort` opens a port on every node. `LoadBalancer` asks the cloud for a public address.
- A Service finds its pods by key-value pairs, so the query and the pairs must match.
- Many `LoadBalancer` Services cost real money, so share an entry point.
