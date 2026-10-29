# Ingress: routing web traffic into the cluster

**Covers:** Ingress, Ingress controller, HTTP, URI, L7

## One front door for many applications {#overview}
Exposing each application with its own public address gets expensive and untidy. Visitors expect to type a name such as `shop.example.com`, and one machine can answer for many names and many paths. Kubernetes has an object, and a matching component, that provide this smart front door for web traffic.

## HTTP {#http}
HTTP, or HyperText Transfer Protocol, is the protocol that web browsers and servers use to exchange pages and data. A request names a host, a path and a method, and the answer carries a status code. Ingress works with exactly these: it reads the host name and the path of each request to decide where to send it.

## HTTPS {#https}
HTTPS is HTTP carried over an encrypted connection. A certificate proves that the server really is the site it claims to be, and the traffic cannot be read or changed on the way. Today nearly all public sites use it. For Ingress, it matters because the encryption is usually ended, or terminated, at the front door, so that everything behind it can be plain HTTP inside the cluster.

## URI {#uri}
A URI, a uniform resource identifier, is a string that names a resource, such as a web address. A web address has parts: a scheme (`https`), a host (`shop.example.com`), a path (`/api/orders`) and sometimes a query (`?page=2`). Ingress rules match on the host and on the path part.

## L7 {#l7}
L7 means layer 7, the application layer of the network stack, where protocols such as HTTP live. A device working at layer 7 can look inside a request and see its host, path and headers. By contrast, a simple load balancer at a lower layer sees only addresses and ports and cannot tell `/orders` from `/payments`. Ingress is a layer 7 tool, which is why it can route by name and path.

## Ingress {#ingress}
An Ingress is an object that defines HTTP routing rules for traffic coming into a cluster from outside, based on things such as the host name or the URL path. For example: requests for `shop.example.com/api` go to the API Service, and requests for `shop.example.com/` go to the web Service. It can also say which certificate to use for HTTPS.

An Ingress object is only a set of rules. It does not move any traffic by itself.

## Ingress controller {#ingress-controller}
An Ingress controller is the component that actually reads the Ingress objects' routing rules and implements them, sending real traffic where the rules say. It is a program, usually a reverse proxy such as NGINX, Traefik or HAProxy, running in the cluster behind a single public address. Kubernetes does not include one by default, so you install one. Without it, Ingress objects have no effect.

When rules change, the controller reloads its configuration, with no interruption to traffic.

## A worked example {#example}
One Ingress that routes two paths and uses HTTPS:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop
spec:
  tls:
    - hosts: [shop.example.com]
      secretName: shop-tls
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: shop-api
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: shop-web
                port:
                  number: 80
```

How a request is handled:

1. A browser asks for `https://shop.example.com/api/orders`. It reaches the single public address of the Ingress controller.
2. The controller ends the HTTPS encryption using the certificate in `shop-tls`.
3. It reads the host (`shop.example.com`) and the path (`/api/orders`). The path starts with `/api`, so the rule for `shop-api` applies.
4. It forwards the request, now plain HTTP, to the `shop-api` Service inside the cluster.
5. A request for `/` or `/about` goes to `shop-web`, as the more general rule.

Count the saving. Ten applications, each exposed with its own cloud load balancer, mean 10 public addresses and 10 bills. With an Ingress controller there is 1 load balancer and 1 address. If a load balancer costs about 20 dollars a month, that is 200 dollars against 20, and certificates are managed in one place, not ten.

Ingress also needs care with order: when two paths match, the longest one wins, which is why `/api` is chosen over `/`.

## A common mistake {#mistake}
Creating Ingress objects without installing an Ingress controller, and then wondering why nothing happens. Install one first. Another mistake is expecting an Ingress to handle non-HTTP traffic, such as a database connection. For that, use a Service of the LoadBalancer type.

## Key points {#key-points}
- An Ingress holds HTTP routing rules by host and path. An Ingress controller carries them out.
- Ingress works at layer 7, so it can look at the host and path of a request.
- HTTPS is usually ended at the Ingress controller, with the certificate kept in the cluster's store for sensitive values.
- One shared entry point is cheaper and tidier than a public address for every application.
