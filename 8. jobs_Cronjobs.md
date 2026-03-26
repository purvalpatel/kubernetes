Description:
------

- A jobs runs a pod that does a task and then stops.
- It’s mean short-lived, one time tasks - not a long running services.

- its like background worker or a cron job that runs only once.

Use cases:
- Data migration scripts
- Batch processing
- Cleanup tasks
- Sending report or a email
- Machine learning training model 


YAML:
```YAML
apiVersion: batch/v1
kind: Job
metadata:
  name: example-job
spec:
  template:
    spec:
      containers:
      - name: hello
        image: busybox
        command: ["echo", "Hello from the Job"]
      restartPolicy: Never
```

## Cronjob:

- Cronjob create job on repeating schedule.

Use case:
- Backups,
- Report generations

YAML:
```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.28
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```
List cron jobs:
```
kubectl get cronjobs -n <namespace>
## output:
NAME                           SCHEDULE       TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob.batch/evidently-cron   */30 * * * *   <none>     False     0        11m             12m
```
##### NAME : cronjob.batch/evidently-cron
- Cronjob name.

##### SCHEDULE :  */30 * * * *
- Run every 30 minutes
  
##### TIMEZONE
No timezone explicitly set, So Kubernetes uses: `UTC (default)`

##### SUSPEND : False
- Whether CronJob is paused
- Value	Meaning
  - False	running normally ✅
  - True	paused (no new jobs) ❌

##### ACTIVE : 0
- Number of currently running jobs
- Value	Meaning
    - 0	no job running now
    - 1+	job currently executing

##### LAST SCHEDULE : 11m
- Last time CronJob triggered

##### AGE : 12m
- How long the CronJob exists


