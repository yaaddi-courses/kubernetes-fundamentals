# Labels and selectors

**Covers:** Label, Labeling, Selector

## Tags that hold a cluster together {#overview}
A cluster may hold hundreds of objects: pods, Services, controllers. To work with some of them and not the others, you need a way to group them. Kubernetes does not use folders or fixed names for this. It uses small tags that you attach yourself, and queries that find objects by their tags. Almost every feature that connects one object to another is built on them.

## Label {#label}
A label is a key-value tag attached to a Kubernetes object, such as `app=shop-web` or `tier=frontend`. Labels have no meaning of their own. You decide what they say, and other objects read them. An object can carry many labels, and many objects can share the same label.

Keys and values are short strings with a limited set of characters, and a key may have a prefix such as `example.com/team`. Labels are meant for identifying information, like the name of the application, its version, its environment or its owner. Long or secret data does not belong in them.

## Labeling {#labeling}
Labeling is the act of attaching labels to objects, and the habit of doing it consistently. You can label at creation time in the file that defines the object, or later with a command such as `kubectl label pod shop-web-1 env=test`. A team that agrees on a few standard keys, such as `app`, `env` and `team`, can find, group and clean up objects easily. A team that labels randomly ends up with objects that no query can find.

## Selector {#selector}
A selector is a query that finds objects by matching their labels. The simplest form is equality: `app=shop-web` matches every object whose `app` label is `shop-web`. Set-based forms allow more: `env in (test, staging)` or `team notin (legacy)`.

Selectors are used both by people and by objects. On the command line, `kubectl get pods -l app=shop-web` lists only the matching pods. Inside the cluster, a Service and a Deployment each carry a selector that tells them which pods are theirs. This is a loose link: the Service does not know the pods by name, only by the labels they carry.

## A worked example {#example}
Six pods, labelled like this:

| Pod | `app` | `env` |
|---|---|---|
| web-1 | shop-web | prod |
| web-2 | shop-web | prod |
| web-3 | shop-web | test |
| db-1 | shop-db | prod |
| db-2 | shop-db | test |
| job-1 | report | prod |

Some queries and what they return:

```bash
kubectl get pods -l app=shop-web
kubectl get pods -l app=shop-web,env=prod
kubectl get pods -l 'env in (test)'
kubectl get pods -l app!=report
```

1. `app=shop-web` returns web-1, web-2 and web-3: 3 pods.
2. Adding `env=prod` with a comma means both must match: web-1 and web-2: 2 pods.
3. `env in (test)` returns web-3 and db-2: 2 pods.
4. `app!=report` returns everything except job-1: 5 pods.

Now a Service whose selector is `app=shop-web`. Its targets are web-1, web-2 and web-3, so a test pod receives production traffic. A better selector is `app=shop-web,env=prod`, with 2 targets.

The same idea lets you act on a group. `kubectl delete pods -l env=test` removes web-3 and db-2 in one command, and leaves production alone. That is why consistent labelling saves time: without it, you would delete 2 pods by name, and with 200 pods, you would need 200 names.

Labels can be changed on a running pod. If you change the `app` label of a failing pod to `debug`, the Service and the controller stop counting it. The controller then starts a replacement, and you can examine the old pod at leisure. This trick is useful for investigating a fault without losing evidence.

## A common mistake {#mistake}
A selector that does not match the labels, often through a small spelling difference, such as `app: shop-web` against `app: shopweb`. The Service then has no targets, with no error message. Check the match with `kubectl get pods --show-labels`. Another mistake is putting changing data, like a timestamp, in a label.

## Key points {#key-points}
- A label is a key-value tag on an object. A selector finds objects by their labels.
- Services and controllers find their pods through selectors, not names.
- Agree on a few standard keys, such as `app`, `env` and `team`, and use them everywhere.
- A comma between conditions means "and".
