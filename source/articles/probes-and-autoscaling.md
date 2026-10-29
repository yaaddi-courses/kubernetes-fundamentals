# Health probes and autoscaling

**Covers:** Liveness probe, Readiness probe, Startup probe, HPA

## Running does not mean working {#overview}
A process can be alive and still useless: stuck in a loop, out of connections, or still loading its data. Kubernetes cannot look inside it, so by default it only knows whether the process exists. To teach it more, you give it small recurring checks, called probes. And once the application is healthy, another question follows: how many copies do you need right now? An autoscaler answers that. Let's take the probes first.

## Liveness probe {#liveness-probe}
A liveness probe asks, again and again, "is this container still alive and working?" If it fails several times in a row, Kubernetes restarts the container. This cures a program that has hung without exiting, such as one caught in a deadlock. The check might be a request to `/healthz` that must return a success code.

Keep it cheap and honest. It should test only the process itself and not other services. Otherwise, a failing database would restart every application in the cluster.

## Readiness probe {#readiness-probe}
A readiness probe asks a different question: "is this container ready to accept real traffic?" Until it passes, the pod stays out of the list of targets of its Service. If it fails later, the pod is taken out again, but not restarted. Use it for a slow start, a temporary overload or a lost connection to something the program depends on.

Without it, a new pod receives requests the moment it starts, before it can answer them, and your users see errors.

## Startup probe {#startup-probe}
What if the application is slow to start? A startup probe gives it time. It is a check that goes first, and while it is going, the other two probes are paused. Once it succeeds, they take over. Without it, a liveness probe might judge a slow starter dead and restart it again and again.

Its patience is the failure threshold times the period. For example, 30 failures at 10 seconds apart allow 300 seconds, or 5 minutes, to start.

## HPA {#hpa}
HPA stands for Horizontal Pod Autoscaler. It adds or removes pods of a Deployment automatically as the load changes, based on a measure such as average CPU use. You give it a minimum, a maximum and a target. It compares the current figure with the target and works out how many pods are needed. "Horizontal" means it changes the number of pods and leaves the size of each pod alone.

## A worked example {#example}
Let's give a web Deployment all three probes:

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

Here is how they act:

1. The container starts and needs 90 seconds to load its data. The startup probe fails 9 times, then succeeds. It had a budget of 300 seconds, so there is no restart.
2. The liveness probe now checks every 10 seconds. If it fails 3 times in a row, which is 30 seconds, the container is restarted.
3. The readiness probe passes only when `/ready` reports that the database is connected. If the connection drops, the pod stops receiving requests, and it returns when the connection does.

Now an autoscaler for the same Deployment:

```bash
kubectl autoscale deployment shop-web --cpu-percent=70 --min=3 --max=10
```

Let's count what it does. Three pods are working at 90% CPU, and the target is 70%. The HPA computes the desired number of pods as 3 × 90 / 70 = 3.86, which it rounds up to 4. When the fourth pod joins, the average falls to about 67%, close to the target. If the load triples, the demand would need about 3 × 270 / 70 = 11.6 pods, but the maximum caps it at 10.

Keep two cautions in mind. The HPA needs CPU requests declared on the pods, because percentages are measured against them. And scaling down is slower than scaling up, by design, to avoid flapping.

## A common mistake {#mistake}
A common slip is using one check for both liveness and readiness, with a dependency inside it. When the database slows down, all pods fail the check at once and restart, which makes things worse. Let liveness check only the process.

Another slip is having no readiness probe, so traffic reaches pods in the middle of a rolling start.

## Key points {#key-points}
- A failed liveness probe restarts the container. A failed readiness probe removes it from traffic, with no restart.
- A startup probe protects slow applications from being killed while they start.
- Keep liveness checks simple and local.
- The HPA changes the number of pods to hold a target, within a minimum and maximum.
