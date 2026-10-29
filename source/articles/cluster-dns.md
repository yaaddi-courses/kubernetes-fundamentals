# Finding things by name: DNS inside a cluster

**Covers:** DNS, Cluster DNS, CoreDNS, MAC

## Names are easier than numbers {#overview}
Even a stable address is a number, and numbers are hard to remember and to put in configuration. People prefer names, and so do programs that you write: "connect to `database`" is clearer than "connect to 10.96.14.20". Kubernetes provides names for everything inside the cluster, using the same technology that names the websites you visit.

## MAC {#mac}
A MAC address, short for media access control address, is the hardware address of a network card, six pairs of characters such as `02:42:ac:11:00:02`. It identifies a device on its local network segment, one layer below the IP address, and it is fixed for the card. You rarely deal with MAC addresses in Kubernetes: the cluster network works with IP addresses, and the MAC addresses stay at the lower level, where the network software handles them. Knowing that they exist explains the lowest steps of "how does a packet find the next device".

## DNS {#dns}
DNS, the Domain Name System, is the service that turns names such as `example.com` into IP addresses. It is a global phone book: a program asks "what is the address of this name?" and gets a number back. The answer may come with an expiry time, after which the program must ask again. Without DNS, you would have to type numbers instead of names into a browser.

## Cluster DNS {#cluster-dns}
Cluster DNS is Kubernetes' built-in naming system, which lets one pod reach another by a stable name, usually the name of a Service, instead of a changing IP address. Every Service gets a name automatically. Pods are configured to ask the cluster's own DNS first, so a lookup for `shop-web` is answered inside the cluster, quickly.

The full name has a pattern: `<service>.<namespace>.svc.cluster.local`. Inside the same namespace, the short name `shop-web` is enough. To reach a Service in another namespace, add it: `shop-web.shop`. The domain `cluster.local` is the default, and can be changed when the cluster is created.

## CoreDNS {#coredns}
CoreDNS is the DNS server that runs inside a Kubernetes cluster so that pods and Services can find each other by name. It runs as a few ordinary pods in the system's own namespace, behind a Service of its own. It watches the API server for Services and pods, and answers questions about them. Names that are not in the cluster, such as `example.com`, are passed on to the outside DNS servers.

Because everything in the cluster relies on it, a failure of CoreDNS shows up as strange errors in many applications at once, with messages such as "could not resolve host".

## A worked example {#example}
Two applications in the namespace `shop`: a web application and a database, each behind a Service.

```bash
kubectl get svc -n shop
kubectl exec -n shop deploy/shop-web -- nslookup shop-db
```

What you would see:

1. The first command lists the Services: `shop-web` and `shop-db`, each with a cluster IP such as `10.96.14.20` and `10.96.20.5`.
2. In the second, a program inside the web pod looks up `shop-db`. Because the pod is in the same namespace, the short name works. CoreDNS answers with `10.96.20.5`.
3. The web application can therefore be configured with `DB_HOST=shop-db`, and not with a number.

The same name from a pod in another namespace, `payments`, must be longer: `shop-db.shop` or the full `shop-db.shop.svc.cluster.local`. A short `shop-db` there would fail with "name not resolved", since the lookup would look in `payments`.

Count the advantage. If the database Service were recreated and got a new IP, `10.96.20.77`, then:
- Configuration with the IP would break, and need editing and redeploying in every application that uses it, perhaps 8 applications.
- Configuration with the name would keep working, since CoreDNS would return the new address on the next lookup, and no application would change.

If lookups seem to fail, check three things in order: is the Service name spelled right and in the right namespace? Does `nslookup kubernetes.default` work from inside a pod? Are the CoreDNS pods running, with `kubectl get pods -n kube-system -l k8s-app=kube-dns`? That sequence finds almost every naming problem.

## A common mistake {#mistake}
Using the short name for a Service in a different namespace. It resolves only inside the caller's own namespace. Add the namespace. Another mistake is hard-coding an IP, which breaks on the next recreation of the Service.

## Key points {#key-points}
- DNS turns names into IP addresses. A MAC address is the lower-level hardware address of a network card.
- Cluster DNS gives every Service a name, such as `shop-db`, or `shop-db.shop` from another namespace.
- CoreDNS is the DNS server inside the cluster. If it fails, many things break at once.
- Use names, not IP addresses, in configuration.
