# Finding things by name: DNS inside a cluster

**Covers:** DNS, Cluster DNS, CoreDNS, MAC

## Names are easier than numbers {#overview}
Think about how you reach a website. You type a name, not a number. Programs benefit in the same way: "connect to `database`" is easier to read, and to put in configuration, than "connect to 10.96.14.20". Even a stable address is only a number. Kubernetes gives every Service a name inside the cluster, using the same technology that names the websites you visit. Let's see how, starting from the bottom layer.

## MAC {#mac}
A MAC address (media access control address) is the hardware address of a network card. It is six pairs of characters, such as `02:42:ac:11:00:02`, and it identifies a device on its local network segment, one layer below the IP address. The card keeps it for life. In Kubernetes you will rarely touch MAC addresses, because the cluster network works with IP addresses, and the network software handles the MAC level for you. It still helps to know they exist, since they explain the lowest step of how a packet finds the next device.

## DNS {#dns}
Above the addresses sits DNS, the Domain Name System. Picture a global phone book. A program asks "what is the address of this name?", such as `example.com`, and gets an IP address back. The answer can carry an expiry time, and after it passes the program must ask again. Without DNS you would type numbers into your browser.

## Cluster DNS {#cluster-dns}
Inside a cluster, the same idea gets a local version. Cluster DNS is Kubernetes' built-in naming. It lets one pod reach another by a stable name, usually the name of a Service, instead of an IP address that may change. Every Service receives a name automatically, and pods are configured to ask the cluster's own DNS first. A lookup for `shop-web` is therefore answered inside the cluster, quickly.

The full name follows a pattern: `<service>.<namespace>.svc.cluster.local`. Inside the same namespace, the short name `shop-web` is enough. To reach a Service in another namespace, add it: `shop-web.shop`. The domain `cluster.local` is the default, and you can change it when you create the cluster.

## CoreDNS {#coredns}
Who answers those questions? CoreDNS does. It is the DNS server inside the cluster, and it lets pods and Services find each other by name. It runs as a few ordinary pods in the system's own namespace, behind a Service of its own. It watches the API server for Services and pods and answers questions about them. Names from outside the cluster, such as `example.com`, are passed on to outside DNS servers.

Because everything depends on it, a CoreDNS failure shows up as strange errors in many applications at once, with messages such as "could not resolve host".

## A worked example {#example}
Let's take two applications in the namespace `shop`: a web application and a database, each behind a Service.

```bash
kubectl get svc -n shop
kubectl exec -n shop deploy/shop-web -- nslookup shop-db
```

Here is what you would see:

1. The first command lists the Services, `shop-web` and `shop-db`, each with a cluster IP, for example `10.96.14.20` and `10.96.20.5`.
2. In the second command, a program inside the web pod looks up `shop-db`. The pod is in the same namespace, so the short name works, and CoreDNS answers with `10.96.20.5`.
3. The web application can therefore be configured with `DB_HOST=shop-db` instead of a number.

Now picture a pod in another namespace, `payments`. For it, the name must be longer: `shop-db.shop` or the full `shop-db.shop.svc.cluster.local`. The short `shop-db` would fail with "name not resolved", because the lookup would search inside `payments`.

Now count the advantage. Suppose the database Service is recreated and receives a new IP, `10.96.20.77`.
- Configuration written with the IP breaks. You would edit and redeploy every application that uses it, perhaps 8 of them.
- Configuration written with the name keeps working. CoreDNS returns the new address on the next lookup, and no application changes.

If lookups fail, check three things in order. Is the Service name spelled correctly, and in the right namespace? Does `nslookup kubernetes.default` work from inside a pod? Are the CoreDNS pods running, which you see with `kubectl get pods -n kube-system -l k8s-app=kube-dns`? This sequence finds almost every naming problem.

## A common mistake {#mistake}
A common slip is using the short name for a Service in a different namespace. It resolves only from the caller's own namespace, so add the namespace to the name.

Another slip is hard-coding an IP address. It works until the Service is recreated, and then it breaks.

## Key points {#key-points}
- DNS turns names into IP addresses. A MAC address is the lower-level hardware address of a network card.
- Cluster DNS gives every Service a name, such as `shop-db`, or `shop-db.shop` from another namespace.
- CoreDNS is the DNS server inside the cluster. If it fails, many things break at once.
- Use names, not IP addresses, in configuration.
