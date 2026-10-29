# Jobs and CronJobs

**Covers:** Kubernetes Job, CronJob

## Work that finishes {#overview}
Most of what runs in a cluster is meant to stay up forever: a web server that should be there tomorrow, and the day after. But some work has a clear end. Convert a file, migrate a database, send a nightly report. If you run that kind of work as an ordinary application, it finishes and Kubernetes helpfully starts it again, forever. Jobs exist for work that is meant to complete.

## Kubernetes Job {#kubernetes-job}
A Kubernetes Job is a controller that runs a task to completion once, and then stops. It creates one or more pods, waits until they finish successfully and records the result. Unlike a long-running controller, it is not meant to keep running, and a pod that exits cleanly is a success, not something to restart.

Useful settings:
- `completions`: how many successful runs are needed in total.
- `parallelism`: how many pods may run at the same time.
- `backoffLimit`: how many times to retry a failing pod before giving up, with growing delays.
- `ttlSecondsAfterFinished`: when to clean up the finished Job and its pods automatically.
- `activeDeadlineSeconds`: a time limit after which the Job is stopped.

If a pod fails, the Job starts a new one, up to the limit, which makes a Job more reliable than running a script on a machine by hand. Finished pods are kept so that you can read their logs.

## CronJob {#cronjob}
A CronJob is a Job that repeats on a schedule that you set, such as a nightly backup, instead of running once. It uses the familiar five-field cron syntax: minute, hour, day of the month, month and day of the week. For example, `0 2 * * *` means 2:00 a.m. every day. At each moment that matches, the CronJob creates a new Job.

Settings that matter:
- `concurrencyPolicy`: what to do if the last run is still going when the next one is due. `Allow` starts another, `Forbid` skips the new one and `Replace` stops the old one.
- `startingDeadlineSeconds`: how late a run may start, if the cluster was busy or down, before it is skipped.
- `successfulJobsHistoryLimit` and `failedJobsHistoryLimit`: how many finished Jobs to keep.

The schedule is read in the time zone of the control plane unless you set `timeZone`, which is a common cause of confusion. Make the task safe to run twice, because a schedule can occasionally run a job twice or miss one.

## A worked example {#example}
A nightly backup as a CronJob:

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

What happens:

1. At 2:00 every night, the CronJob creates a Job from the template.
2. The Job starts one pod, which runs `backup.sh`. If it exits with success, the Job is done.
3. If the pod fails, the Job retries up to 2 more times (`backoffLimit: 2`).
4. If last night's run is still going at 2:00, `Forbid` skips the new one.
5. The three most recent successful Jobs are kept, so that their logs can be read.

Count the retries and the history. With a `backoffLimit` of 2, a task that always fails runs 3 times in total (the first attempt and 2 retries), and the Job is then marked failed. With a history limit of 3, after a week, there are only 3 successful Jobs left, and not 7, so the cluster stays tidy.

The scheduling arithmetic: `0 2 * * *` runs 365 times in a year. If a backup takes 20 minutes, the cluster spends about 365 x 20 = 7,300 minutes a year, or about 122 hours, on it, which is not much. If someone changes the schedule to `*/5 * * * *`, which is every 5 minutes, it runs 288 times a day, and with a 20-minute task the runs would overlap. `Forbid` stops them piling up.

Run a one-off Job by hand from a CronJob to test it: `kubectl create job --from=cronjob/nightly-backup test-backup`.

## A common mistake {#mistake}
Running a finite task as a long-running controller, which restarts it forever when it finishes. Use a Job. Another mistake is writing a CronJob task that cannot be safely repeated, such as one that adds records without checking whether they exist, and then having a duplicate run.

## Key points {#key-points}
- A Job runs a task to completion, retries on failure up to a limit and then stops.
- A CronJob creates Jobs on a cron schedule. Choose `concurrencyPolicy` and mind the time zone.
- Keep a short history of finished Jobs so that logs can be read.
- Make scheduled tasks safe to run twice.
