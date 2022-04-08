# Introduction to Kubernetes

Editors: **Kaan Keskin, Sezen Erdem**

Date: November 2021

Available at: https://github.com/kaan-keskin/introduction-to-kubernetes

**Resources:**

> - Kubernetes Documentation - https://kubernetes.io/docs/home/
> - Kubernetes in Action - Marko Lukša - Manning Publications
> - Kubernetes Fundamentals (LFS258) - Timothy Serewicz - The Linux Foundation
> - Kubernetes for Developers (LFD259) - Timothy Serewicz - The Linux Foundation
> - Kubernetes Lecture Notes - California Institute of Technology
> - Certified Kubernetes Application Developer (CKAD) Study Guide - Benjamin Muschko - O'Reilly Media
> - Getting Started with Kubernetes - Sander van Vugt - Addison-Wesley Professional

**LEGAL NOTICE: This document is created for educational purposes, and it can not be used for any commercial intentions. If you find this document useful in any means please support the original authors for ethical reasons.** 

[Return to the README page.](README.md)

# Jobs

 Pod is meant for the continuous operation of an application. You will want to deploy the application with a specific version and then keep it running without interrupts if possible.

A Job is a Kubernetes primitive with a different goal. It runs functionality until a specified number of completions has been reached, making it a good fit for one-time operations like import/export data processes or I/O-intensive processes with a finite end. The actual work managed by a Job is still running inside of a Pod. Therefore, you can think of a Job as a higher-level coordination instance for Pods executing the workload.

Upon completion of a Job and its Pods, Kubernetes does not automatically delete the objects—they will stay until they’re explicity deleted. Keeping those objects helps with debugging the command run inside of the Pod and gives you a chance to inspect the logs.

Kubernetes supports an auto-cleanup mechanism for Jobs and their controlled Pods by specifying the attribute spec.ttlSecondsAfterFinished. Be aware that the feature is still in an alpha state and needs to be enabled explicitly for a cluster by setting a feature flag.

Just as we may need to redesign our applications to be decoupled, we may also consider that microservices may not need to run all the time. The use of Jobs and CronJobs can further assist with implementing decoupled and transient microservices.

Jobs are part of the batch API group. They are used to run a set number of pods to completion. If a pod fails, it will be restarted until the number of completion is reached.

While they can be seen as a way to do batch processing in Kubernetes, they can also be used to run one-off pods. A Job specification will have a parallelism and a completion key. If omitted, they will be set to one. If they are present, the parallelism number will set the number of pods that can run concurrently, and the completion number will set how many pods need to run successfully for the Job itself to be considered done. Several Job patterns can be implemented, like a traditional work queue. 

Cronjobs work in a similar manner to Linux jobs, with the same time syntax. There are some cases where a job would not be run during a time period or could run twice; as a result, the requested Pod should be idempotent. 

An option spec field is .spec.concurrencyPolicy, which determines how to handle existing jobs, should the time segment expire. If set to Allow, the default, another concurrent job will be run. If set to Forbid, the current job continues and the new job is skipped. A value of Replace cancels the current job and starts a new job in its place.

Kubernetes includes support for creation of pods that terminates after process running inside finishes successfully and not restarted again.

In the event of a node failure, the pods on that node that are managed by a Job will be rescheduled to other nodes.

In the event of a failure of the process itself (when the process returns an error exit code), the Job can be configured to either restart the container or not.

* Run multiple pod instances in a Job
    * Run job pods sequentially
    * Run job pods in parallel
    * Scale a Job
* Limit the time allowed for a Job pod to complete

## Creating and Inspecting Jobs

To create a Job imperatively, simply use the create job command. If the provided image doesn’t run any commands, you may want to append a command to be executed in the corresponding Pod. The following command creates a Job that runs an iteration process. For each iteration of the loop, a variable named counter is incremented. The command execution finishes after reaching the counter value 3:

```shell
$ kubectl create job counter --image=nginx -- /bin/sh -c 'counter=0; \
  while [ $counter -lt 3 ]; do counter=$((counter+1)); echo "$counter"; \
  sleep 3; done;'

job.batch/counter created
```

The output of listing the Job shows the current number of completions and the expected number of completions. The default number of completions is 1. This means if the Pod executing the command was successful, a Job is considered completed. As you can see in the following terminal output, a Job uses a single Pod by default to perform the work. The corresponding Pod can be identified by name—it uses the Job name as a prefix in its own name:

```shell
$ kubectl get jobs

NAME      COMPLETIONS   DURATION   AGE
counter   0/1           13s        13s

$ kubectl get jobs

NAME      COMPLETIONS   DURATION   AGE
counter   1/1           15s        19s

$ kubectl get pods

NAME            READY   STATUS      RESTARTS   AGE
counter-z6kdj   0/1     Completed   0          51s
```

To verify the correct behavior of the Job, you can download the logs of the Pod. As expected, the output renders the counter for each iteration:

```shell
$ kubectl logs counter-z6kdj
1
2
3
```

If you’d rather go the declarative route by creating a YAML manifest, you can define a Job object via yaml file. 

A Job executing a loop command:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: counter
spec:
  template:
    spec:
      containers:
      - name: counter
        image: nginx
        command:
        - /bin/sh
        - -c
        - counter=0; while [ $counter -lt 3 ]; do counter=$((counter+1)); \
          echo "$counter"; sleep 3; done;
      restartPolicy: Never
```

Here is an example Job config. It computes π to 2000 places and prints it out. It takes around 10s to complete.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```

```shell
$ kubectl apply -f job.yaml
```

## Job Operation Types

The default behavior of a Job is to run the workload in a single Pod and expect one successful completion. That’s what Kubernetes calls a non-parallel Job. Internally, those parameters are tracked by the attributes spec.template.spec.completions and spec.template.spec.parallelism, each with the assigned value 1. The following command renders the parameters of the Job we created earlier:

```shell
$ kubectl get jobs counter -o yaml | grep -C 1 "completions"

...
  completions: 1
  parallelism: 1
...
```

You can tweak any of those parameters to fit the needs of your use case. Say you’d expect the workload to complete successfully multiple times; then you’d increase the value of spec.template.spec.completions to at least 2. Sometimes, you’ll want to execute the workload by multiple pods in parallel. In those cases, you’d bump up the value assigned to spec.template.spec.parallelism. This is referred to as a parallel job. Remember that you can use any combination of assigned values for both attributes. Table below summarizes the different use cases.

Configuration for different Job operation types:

| Type | spec.completions | spec.parallelism | Description |
| :--: | :--: | :--: | :--: |
| Non-parallel with one completion count | 1 | 1 | Completes as soon as its Pod terminates successfully. |
| Parallel with a fixed completion count | >= 1 | >= 1 | Completes when specified number of tasks finish successfully. |
| Parallel with worker queue | unset | >= 1 | Completes when at least one Pod has terminated successfully and all Pods are terminated. |

## Restart Behavior

The spec.backoffLimit attribute determines the number of retries a Job attempts to successfully complete the workload until the executed command finishes with an exit code 0. The default is 6, which means it will execute the workload 6 times before the Job is considered unsuccessful.

The Job manifest needs to explicitly declare the restart policy by using spec.template.spec.restartPolicy. The default restart policy of a Pod is Always, which tells the Kubernetes scheduler to always restart the Pod even if the container exits with a zero exit code. The restart policy of a Job can only be OnFailure or Never.

### Restarting the Container on Failure

Figure below shows the behavior of a Job that uses the restart policy OnFailure. Upon a container failure, this policy will simply rerun the container.

Restart policy onFailure:

<img src=".\images\job-restart-policy-onFailure.png"/>

### Starting a New Pod on Failure

Figure below shows the behavior of a Job that uses the restart policy Never. This policy does not restart the container upon a failure. It starts a new Pod instead.

Restart policy Never:

<img src=".\images\job-restart-policy-never.png"/>

# CronJobs

Job resources run their pods immediately when you create the Job resource. But many batch jobs need to be run at a specific time in the future or repeatedly in the specified interval. In Linux- and UNIX-like operating systems, these jobs are better known as cron jobs. Kubernetes supports them, too. A CronJob creates Jobs on a repeating schedule.

A Job represents a finite operation. Once the operation could be executed successfully, the work is done and the Job will create no more Pods. A CronJob is essentially a Job, but it’s run periodically based a schedule; however, it will continue to create a new Pod when it’s time to run the task. The schedule can be defined with a cron-expression you may already know from Unix cron jobs. Figure below shows a CronJob that executes every hour. For every execution, the CronJob creates a new Pod that runs the task and finishes with a zero or non-zero exit code.

A cron job in Kubernetes is configured by creating a CronJob resource. At the configured time, Kubernetes will create a Job resource according to the Job template configured in the CronJob object. When the Job resource is created, one or more pod replicas will be created and started according to the Job’s pod template.

Executing a Job based on a schedule:

<img src=".\images\executing-job-schedule.png"/>

## Creating and Inspecting Jobs

You can use the imperative create cronjob command to create a new CronJob. The following command schedules the CronJob to run every minute. The Pod created for every execution renders the current date to standard output using the Unix echo command:

```shell
$ kubectl create cronjob current-date --schedule="* * * * *" --image=nginx -- /bin/sh -c 'echo "Current date: $(date)"'

cronjob.batch/current-date created
```

If you list the existing CronJob with the get cronjobs command, you will see the schedule, the last scheduled execution, and whether the CronJob is currently active. It’s easy to match Pods managed by a CronJob. You can simply identify them by the name prefix. In this case, the prefix is current-date-:

```shell
$ kubectl get cronjobs

NAME           SCHEDULE    SUSPEND   ACTIVE   LAST SCHEDULE   AGE
current-date   * * * * *   False     0        <none>>         28s

$ kubectl get cronjobs

NAME           SCHEDULE    SUSPEND   ACTIVE   LAST SCHEDULE   AGE
current-date   * * * * *   False     1        14s             53s

$ kubectl get cronjobs

NAME           SCHEDULE    SUSPEND   ACTIVE   LAST SCHEDULE   AGE
current-date   * * * * *   False     0        19s             58s

$ kubectl get pods

NAME                            READY   STATUS              RESTARTS   AGE
current-date-1598651700-2xgn9   0/1     Completed           0          63s
current-date-1598651760-rt6c6   0/1     ContainerCreating   0          1s
```

A CronJob printing the current date:

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: current-date
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: current-date
            image: nginx
            args:
            - /bin/sh
            - -c
            - 'echo "Current date: $(date)"'
          restartPolicy: OnFailure
```

This example CronJob manifest prints the current time and a hello message every minute:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            imagePullPolicy: IfNotPresent
            command:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

```shell
kubectl apply -f cronjob.yaml
```

## Configuring Retained Job History

Even after completing a task in a Pod controlled by a CronJob, it will not be deleted automatically. Keeping a historical record of Pods can be tremendously helpful for troubleshooting failed workloads or inspecting the logs. By default, a CronJob retains the last three successful Pods and the last failed Pod:

```shell
$ kubectl get cronjobs current-date -o yaml | grep successfulJobsHistoryLimit:

  successfulJobsHistoryLimit: 3

$ kubectl get cronjobs current-date -o yaml | grep failedJobsHistoryLimit:

  failedJobsHistoryLimit: 1
```

In order to reconfigure the job retention history limits, set new values for the attributes spec.successfulJobsHistoryLimit and spec.failedJobsHistoryLimit. 

Example below keeps the last five successful executions and the last three failed executions.

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: current-date
spec:
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 3
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: current-date
            image: nginx
            args:
            - /bin/sh
            - -c
            - 'echo "Current date: $(date)"'
          restartPolicy: OnFailure
```

