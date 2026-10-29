# Rolling updates and rollbacks

**Covers:** Rolling update, Rollback

## Shipping a new version without a gap {#overview}
Think of the old way to release software: stop the application, install the new version and start it again. For those minutes, users saw an error page. When several copies of the application are alive, there is a better way. You change them one at a time, so that enough copies are always serving. And since a new version can be bad, you also need a quick way back. This article covers both.

## Rolling update {#rolling-update}
A rolling update replaces old pods with new ones a few at a time instead of all at once, so the application stays available during the release. A Deployment does this by default. When you change the image, it starts some new pods, waits until they are ready, stops some old ones and repeats until every pod is new.

Two settings control the speed. `maxSurge` is how many extra pods may exist above the wanted number during the update. `maxUnavailable` is how many may be missing. By default, each is 25%. A readiness check matters a lot here, because the Deployment counts a new pod as ready only when it passes. A broken version therefore stops the roll before it replaces everything.

## Rollback {#rollback}
A rollback returns a Deployment to an earlier working version after a bad update. The Deployment keeps old ReplicaSets, one for each version, so going back means scaling the old one up and the new one down, again as a rolling update. It is quick, because the images are usually already on the nodes.

By default, the last 10 revisions are kept, controlled by `revisionHistoryLimit`. A version older than that is out of reach.

## A worked example {#example}
Let's say a Deployment has 4 pods of version 1.4. With `maxSurge: 1` and `maxUnavailable: 0`, you release 1.5.

```bash
kubectl set image deployment/shop-web web=shop/web:1.5
kubectl rollout status deployment/shop-web
```

The steps go like this:

1. Start 1 new pod (version 1.5). Now there are 5 pods, 4 old and 1 new. That is the surge of 1.
2. When the new pod is ready, stop 1 old one. Now there are 3 old and 1 new.
3. Start another new one, giving 3 old and 2 new, 5 in total. When it is ready, stop an old one, leaving 2 old and 2 new.
4. Continue until 0 old and 4 new remain.

At every moment at least 4 pods were ready, so the application never fell below its full capacity. With `maxUnavailable: 0` you lose no capacity, and the price is one extra pod at a time.

Now suppose version 1.5 has a bug that the readiness check did not catch. You go back:

```bash
kubectl rollout history deployment/shop-web
kubectl rollout undo deployment/shop-web
```

The history lists the revisions, such as 1 for 1.4 and 2 for 1.5. The undo brings back revision 1 as a new rolling update, at the same safe pace. You can also choose a revision: `kubectl rollout undo deployment/shop-web --to-revision=1`.

Let's count the exposure. If the bug breaks every request and the rollout stalls halfway because the readiness check fails, the bad pods never become ready and receive no traffic. Compare the all-at-once way: all 4 of 4 pods are replaced, and every user is affected until you fix it.

If a rollout stalls, `kubectl rollout status` waits and eventually reports a failure, while the old pods keep serving. That is the moment to undo it.

## A common mistake {#mistake}
A common slip is rolling out without a readiness check. New pods count as ready immediately, so a broken version can replace all the old ones within seconds.

Another slip is a change that is not backward compatible, such as a database change that version 1.4 cannot read. For a short time both versions are alive together, so both must work with the same data.

## Key points {#key-points}
- A rolling update replaces pods gradually, so the application stays up. `maxSurge` and `maxUnavailable` control the pace.
- A readiness check is what lets a bad version stop the update.
- A rollback goes back to an earlier revision, using the old ReplicaSets that the Deployment kept.
- During an update, old and new versions live side by side.
