# Health probes and autoscaling

**Covers:** Liveness probe, Readiness probe, Startup probe, HPA

## Running does not mean working {#overview}
A process can be alive and still useless: stuck in a loop, out of connections, or still loading its data. Kubernetes cannot see inside it, so by default it only knows whether the process exists. You teach it more by giving it small checks to run, called probes. And once the application is healthy, a second question comes up: how many copies are needed right now? An autoscaler answers it.

## Liveness probe {#liveness-probe}
A liveness probe is a recurring check that asks "is this container still alive and working?" If it fails several times in a row, Kubernetes restarts the container. It is the cure for a program that has hung without exiting, such as a deadlock. The check might be a request to `/healthz` that must return a success code.

Keep it cheap and honest. It should test only the process itself, not other services, or a failure of the database would restart every application in the cluster.

## Readiness probe {#readiness-probe}
A readiness probe is a check that asks "is this container ready to accept real traffic?" Until it passes, the pod is left out of the list of targets of its Service. If it later fails, the pod is taken out again, but is not restarted. It is the right tool for a slow start, a temporary overload or a lost connection to a dependency.

Without it, a new pod gets requests the moment it starts, before it can answer them, and users see errors.

## Startup probe {#startup-probe}
A startup probe is a check that runs first and gives a slow application time to start. While it is running, the other two probes are paused. Once it succeeds, they take over. Without it, a liveness probe might judge a slow starter as dead and restart it again and again.

Its patience is set by the failure threshold times the period. For example, 30 failures at 10 seconds allow 300 seconds, or 5 minutes, to start.

## HPA {#hpa}
HPA stands for Horizontal Pod Autoscaler. It adds or removes pods of a Deployment automatically as load changes, based on a measure such as average CPU use. You set a minimum, a maximum and a target. The HPA compares the current figure to the target and computes how many pods are needed. "Horizontal" means that it changes the number of pods, not the size of each.

## A worked example {#example}
A Deployment for a web application, with all three probes:

```yaml
containers:
  - name: web
    image: shop/web:1.4
    startupProbe:
      httpGet: {path: /healthz, port: 8000}
      periodSeconds: 10
      failureThreshold: 30
    livenessProbe:
      httpGet: {path: /healthz, port: 8000}
      periodSeconds: 10
      failureThreshold: 3
    readinessProbe:
      httpGet: {path: /ready, port: 8000}
      periodSeconds: 5
```

How they act:

1. The container starts and needs 90 seconds to load its data. The startup probe fails 9 times, then succeeds. It had a budget of 300 seconds, so there is no restart.
2. The liveness probe now runs every 10 seconds. If it fails 3 times in a row, which is 30 seconds, the container is restarted.
3. The readiness probe passes only when `/ready` reports that the database is connected. If the database connection drops, the pod stops receiving requests, and comes back when the connection returns.

An autoscaler for the same Deployment:

```bash
kubectl autoscale deployment shop-web --cpu-percent=70 --min=3 --max=10
```

Count what it does. Three pods are running at 90% CPU. The target is 70%. The HPA computes the desired number of pods as 3 × 90 / 70 = 3.86, and rounds up to 4. After the fourth pod joins, the average falls to about 67%, near the target. If load triples, the demand would need about 3 × 270 / 70 = 11.6 pods, but the maximum caps it at 10.

Two cautions. The HPA needs CPU requests set on the pods, since percentages are measured against them. And scaling down is slower than scaling up, by design, to avoid flapping.

## A common mistake {#mistake}
Using the same check for liveness and readiness, with a dependency in it. When the database slows down, all pods fail the check at once and restart, which makes it worse. Make liveness check only the process. Another mistake is no readiness probe, so that traffic reaches pods during a rolling start.

## Key points {#key-points}
- Liveness failure restarts the container. Readiness failure removes it from traffic, without a restart.
- A startup probe protects slow applications from being killed during start.
- Keep liveness checks simple and local.
- The HPA changes the number of pods to hold a target, within a minimum and maximum.
