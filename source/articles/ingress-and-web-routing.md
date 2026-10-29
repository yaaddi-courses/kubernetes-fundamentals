# Ingress: routing web traffic into the cluster

**Covers:** Ingress, Ingress controller, HTTP, URI, L7

## One front door for many applications {#overview}
Imagine you host ten web applications in one cluster. Giving each its own public address is expensive and untidy. Visitors only want to type a name such as `shop.example.com`, and one machine can answer for many names and many paths. Kubernetes offers an object and a matching component that together act as a smart front door for web traffic. To understand them, let's first look at the parts of a web request that they read.

## HTTP {#http}
HTTP, the HyperText Transfer Protocol, is how web browsers and servers exchange pages and data. A request names a host, a path and a method, and the answer carries a status code. Ingress works with exactly these pieces: it reads the host name and the path of each request to decide where to send it.

## HTTPS {#https}
HTTPS is HTTP carried over an encrypted connection. A certificate proves that the server really is the site it claims to be, and nobody can read or change the traffic on the way. Nearly all public sites use it today. For Ingress it matters in one particular way: the encryption is usually ended, or terminated, at the front door, so everything behind it can be plain HTTP inside the cluster.

## URI {#uri}
A URI, a uniform resource identifier, is the string that names a resource, such as a web address. Take a web address apart and you find a scheme (`https`), a host (`shop.example.com`), a path (`/api/orders`) and sometimes a query (`?page=2`). Ingress rules match on the host and on the path.

## L7 {#l7}
Networks are described in layers, and L7 is layer 7, the application layer, where protocols such as HTTP live. A device working at that layer can look inside a request and see its host, path and headers. A simple load balancer at a lower layer sees only addresses and ports, so it cannot tell `/orders` from `/payments`. Ingress works at layer 7, which is why it can route by name and path.

## Ingress {#ingress}
Now we can say what an Ingress is. It is an object that holds HTTP routing rules for traffic entering the cluster from outside, based on things such as the host name or the URL path. For example: requests for `shop.example.com/api` go to the API Service, and requests for `shop.example.com/` go to the web Service. It can also name the certificate to use for HTTPS.

Keep one point in mind. An Ingress object is only a set of rules, and it does not move any traffic by itself.

## Ingress controller {#ingress-controller}
Something has to carry out the rules, and that is the Ingress controller. It reads the Ingress objects and sends real traffic where the rules say. It is a program, usually a reverse proxy such as NGINX, Traefik or HAProxy, working in the cluster behind a single public address. Kubernetes does not include one by default, so you install one. Without it, Ingress objects have no effect.

When the rules change, the controller reloads its configuration without interrupting traffic.

## A worked example {#example}
Let's read one Ingress that routes two paths and uses HTTPS:

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

Here is how one request is handled:

1. A browser asks for `https://shop.example.com/api/orders`. It reaches the single public address of the Ingress controller.
2. The controller ends the HTTPS encryption using the certificate in `shop-tls`.
3. It reads the host (`shop.example.com`) and the path (`/api/orders`). The path starts with `/api`, so the rule for `shop-api` applies.
4. It forwards the request, now plain HTTP, to the `shop-api` Service inside the cluster.
5. A request for `/` or `/about` goes to `shop-web`, as the more general rule.

Now count the saving. Ten applications, each exposed with its own cloud load balancer, mean 10 public addresses and 10 bills. With an Ingress controller there is 1 load balancer and 1 address. If a load balancer costs about 20 dollars a month, that is 200 dollars against 20, and you manage certificates in one place instead of ten.

Order matters too. When two paths match, the longest one wins, which is why `/api` is chosen over `/`.

## A common mistake {#mistake}
A common slip is creating Ingress objects without installing an Ingress controller, and then wondering why nothing happens. Install a controller first.

Another slip is expecting an Ingress to handle traffic that is not HTTP, such as a database connection. For that, use a Service of the LoadBalancer type.

## Key points {#key-points}
- An Ingress holds HTTP routing rules by host and path. An Ingress controller carries them out.
- Ingress works at layer 7, so it can look at the host and path of a request.
- HTTPS is usually ended at the Ingress controller, with the certificate kept in the cluster's store for sensitive values.
- One shared entry point is cheaper and tidier than a public address for every application.
