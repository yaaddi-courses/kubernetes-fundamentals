# Labels and selectors

**Covers:** Label, Labeling, Selector

## Tags that hold a cluster together {#overview}
A cluster may hold hundreds of objects: pods, Services, controllers. Suppose you want to work with some of them and leave the others alone. How do you say which ones? Kubernetes does not use folders or fixed names. It uses small tags that you attach yourself, and queries that find objects by their tags. Almost every feature that links one object to another is built on this pair, so it is worth learning well.

## Label {#label}
A label is a key-value tag on an object, such as `app=shop-web` or `tier=frontend`. Labels carry no meaning of their own. You decide what they say, and other objects read them. An object can have many labels, and many objects can share the same label.

Keys and values are short strings with a limited set of characters, and a key may have a prefix such as `example.com/team`. Use labels for identifying information: the application's name, its version, its environment or its owner. Long or secret data does not belong in them.

## Labeling {#labeling}
Labeling means attaching labels to objects, and doing it consistently. You can label when you create the object, in its definition file, or later with a command such as `kubectl label pod shop-web-1 env=test`. A team that agrees on a few standard keys, such as `app`, `env` and `team`, can find, group and clean up objects with ease. A team that labels at random ends up with objects that no query can find.

## Selector {#selector}
A selector is the query that finds objects by their labels. The most basic form is equality: `app=shop-web` matches every object whose `app` label is `shop-web`. Set-based forms say more: `env in (test, staging)` or `team notin (legacy)`.

People and objects both use selectors. On the command line, `kubectl get pods -l app=shop-web` lists only the matching pods. Inside the cluster, a Service and a Deployment each carry a selector that tells them which pods are theirs. This is a loose link: the Service does not know the pods by name, only by the labels they carry.

## A worked example {#example}
Let's label six pods like this:

| Pod | `app` | `env` |
|---|---|---|
| web-1 | shop-web | prod |
| web-2 | shop-web | prod |
| web-3 | shop-web | test |
| db-1 | shop-db | prod |
| db-2 | shop-db | test |
| job-1 | report | prod |

Now try some queries:

```bash
kubectl get pods -l app=shop-web
kubectl get pods -l app=shop-web,env=prod
kubectl get pods -l 'env in (test)'
kubectl get pods -l app!=report
```

1. `app=shop-web` returns web-1, web-2 and web-3: 3 pods.
2. Adding `env=prod` with a comma means both conditions must match, so you get web-1 and web-2: 2 pods.
3. `env in (test)` returns web-3 and db-2: 2 pods.
4. `app!=report` returns everything except job-1: 5 pods.

Now imagine a Service whose selector is `app=shop-web`. Its targets are web-1, web-2 and web-3, so a test pod receives production traffic. A better selector is `app=shop-web,env=prod`, which has 2 targets.

The same idea lets you act on a whole group. `kubectl delete pods -l env=test` removes web-3 and db-2 in one command and leaves production alone. That is why consistent labelling saves time: without it you would name 2 pods by hand, and with 200 pods you would need 200 names.

Labels can also change on a live pod. Say a pod is misbehaving and you change its `app` label to `debug`. The Service and the controller stop counting it, and the controller starts a replacement. You can now examine the old pod at leisure. This trick lets you investigate a fault without losing the evidence.

## A common mistake {#mistake}
A common slip is a selector that does not match the labels, often through a small spelling difference such as `app: shop-web` against `app: shopweb`. The Service then has no targets, and no error message tells you so. Check the match with `kubectl get pods --show-labels`.

Another slip is putting changing data, like a timestamp, in a label. Labels are for identity, not for logging.

## Key points {#key-points}
- A label is a key-value tag on an object. A selector finds objects by their labels.
- Services and controllers find their pods through selectors, not names.
- Agree on a few standard keys, such as `app`, `env` and `team`, and use them everywhere.
- A comma between conditions means "and".
