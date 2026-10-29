# Jobs and CronJobs

**Covers:** Kubernetes Job, CronJob

## Work that finishes {#overview}
Most of what lives in a cluster is meant to stay up: a web server that should be there tomorrow, and the day after. But some work has a clear end. Convert a file, migrate a database, send a nightly report. If you start that kind of work as an ordinary application, it finishes, and Kubernetes helpfully starts it again, forever. Jobs exist for work that is meant to complete, and CronJobs repeat such work on a schedule.

## Kubernetes Job {#kubernetes-job}
A Kubernetes Job is a controller that carries a task through to completion once, and then stops. It creates one or more pods, waits until they finish successfully and records the result. A pod that exits cleanly counts as a success, not as something to restart, which is the opposite of how a long-lived controller behaves.

A Job has a few settings you will meet often:
- `completions`: how many successful runs are needed in total.
- `parallelism`: how many pods may work at the same time.
- `backoffLimit`: how many times to retry a failing pod before giving up, with growing delays.
- `ttlSecondsAfterFinished`: when to clean up the finished Job and its pods automatically.
- `activeDeadlineSeconds`: a time limit after which the Job is stopped.

If a pod fails, the Job starts a new one, up to the limit. That makes a Job more reliable than starting a script on a machine by hand. Finished pods are kept so that you can read their logs.

## CronJob {#cronjob}
What if the task must repeat, like a nightly backup? A CronJob is a Job on a schedule that you choose. It uses the familiar five-field cron syntax: minute, hour, day of the month, month and day of the week. For example, `0 2 * * *` means 2:00 a.m. every day. At each matching moment, the CronJob creates a new Job.

These settings matter most:
- `concurrencyPolicy`: what to do if the last run is still going when the next one is due. `Allow` starts another, `Forbid` skips the new one and `Replace` stops the old one.
- `startingDeadlineSeconds`: how late a run may start, if the cluster was busy or down, before it is skipped.
- `successfulJobsHistoryLimit` and `failedJobsHistoryLimit`: how many finished Jobs to keep.

One trap is the time zone. The schedule is read in the time zone of the control plane unless you give a `timeZone`, and this is a common cause of confusion. Also make the task safe to repeat, because a schedule can occasionally start a job twice or miss one.

## A worked example {#example}
Let's write a nightly backup as a CronJob:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: backup
              image: shop/backup:1.0
              command: ["./backup.sh"]
```

Here is what happens:

1. At 2:00 every night, the CronJob creates a Job from the template.
2. The Job starts one pod, which executes `backup.sh`. If it exits with success, the Job is done.
3. If the pod fails, the Job retries up to 2 more times (`backoffLimit: 2`).
4. If last night's backup is still going at 2:00, `Forbid` skips the new one.
5. The three most recent successful Jobs are kept, so their logs can be read.

Let's count the retries and the history. With a `backoffLimit` of 2, a task that always fails is tried 3 times in total, the first attempt and 2 retries, and then the Job is marked failed. With a history limit of 3, after a week only 3 successful Jobs remain, not 7, so the cluster stays tidy.

Now the schedule arithmetic. `0 2 * * *` fires 365 times in a year. If a backup takes 20 minutes, the cluster spends about 365 x 20 = 7,300 minutes a year on it, or about 122 hours, which is not much. Suppose someone changes the schedule to `*/5 * * * *`, which means every 5 minutes. That fires 288 times a day, and with a 20-minute task the runs would overlap. `Forbid` stops them from piling up.

To test a CronJob without waiting for the night, create a one-off Job from it by hand: `kubectl create job --from=cronjob/nightly-backup test-backup`.

## A common mistake {#mistake}
A common slip is starting a finite task as a long-lived controller, which restarts it forever when it finishes. Use a Job for work that ends.

Another slip is writing a CronJob task that cannot be repeated safely, for example one that adds records without checking whether they already exist. The first duplicate run then leaves you with duplicate data.

## Key points {#key-points}
- A Job carries a task to completion, retries on failure up to a limit and then stops.
- A CronJob creates Jobs on a cron schedule. Choose `concurrencyPolicy` and mind the time zone.
- Keep a short history of finished Jobs so that logs can be read.
- Make scheduled tasks safe to repeat.
