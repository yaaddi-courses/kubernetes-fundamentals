# Rolling updates and rollbacks

**Covers:** Rolling update, Rollback

## Shipping a new version without a gap {#overview}
The old way to release was to stop the application, install the new version, and start it again. During that time, users saw an error page. With several copies of the application running, there is a better way: change them one at a time, so that enough copies are always serving. And if the new version is bad, you need a fast way back.

## Rolling update {#rolling-update}
A rolling update replaces old pods with new ones a few at a time, instead of all at once, so that the application stays available during the release. A Deployment does it by default. When you change the image, it starts some new pods, waits until they are ready, and then stops some old ones, repeating until all are new.

Two settings control the speed. `maxSurge` is how many extra pods may exist above the wanted number during the update. `maxUnavailable` is how many may be missing. By default, each is 25%. A readiness check is important here: the Deployment counts a new pod as ready only when it passes, so a broken version stops the roll before it replaces everything.

## Rollback {#rollback}
A rollback returns a Deployment to a previous working version after a bad update. The Deployment keeps old ReplicaSets, one for each version, so going back means scaling the old one up and the new one down, again as a rolling update. It is quick, since the images are usually already on the nodes.

By default, the last 10 revisions are kept, set by `revisionHistoryLimit`. A rollback to a version that is older than that is not possible.

## A worked example {#example}
A Deployment has 4 pods of version 1.4. With `maxSurge: 1` and `maxUnavailable: 0`, you release 1.5.

```bash
kubectl set image deployment/shop-web web=shop/web:1.5
kubectl rollout status deployment/shop-web
```

The steps:

1. Start 1 new pod (version 1.5). Now there are 5 pods: 4 old and 1 new. That is the surge of 1.
2. When the new pod is ready, stop 1 old one. Now 3 old and 1 new.
3. Start another new one: 3 old, 2 new, 5 in total. When ready, stop an old one: 2 old, 2 new.
4. Continue until 0 old and 4 new remain.

At every moment at least 4 pods were ready, so the application never fell below its full capacity. With `maxUnavailable: 0`, there is no loss of capacity, and the price is the one extra pod at a time.

Now version 1.5 turns out to have a bug that the readiness probe did not catch. You go back:

```bash
kubectl rollout history deployment/shop-web
kubectl rollout undo deployment/shop-web
```

The history lists the revisions, such as 1 for 1.4 and 2 for 1.5. The undo brings back revision 1 as a new rolling update, with the same safe pace. You can also pick one: `kubectl rollout undo deployment/shop-web --to-revision=1`.

Count the exposure. If the bug breaks 100% of requests, and the rollout was stopped halfway by a failing readiness check, only the 2 new pods out of 4 were affected, or 0 requests if they never became ready. Compare the all-at-once way: 4 of 4 pods replaced, and 100% of users affected until fixed.

If a rollout stalls, `kubectl rollout status` waits and eventually reports a failure, and the old pods keep serving. Then undo it.

## A common mistake {#mistake}
Rolling out without a readiness probe. New pods count as ready immediately, so a broken version replaces all the old ones within seconds. Another mistake is a change that is not backward compatible, such as a database change that version 1.4 cannot read. For a short time, both versions run together, so both must work with the same data.

## Key points {#key-points}
- A rolling update replaces pods gradually, so the application stays up. `maxSurge` and `maxUnavailable` set the pace.
- A readiness probe is what lets a bad version stop the update.
- A rollback goes back to an earlier revision, using the old ReplicaSets that the Deployment kept.
- During an update, old and new versions run side by side.
