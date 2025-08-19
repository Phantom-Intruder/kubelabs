## Using New Relic with Kubernetes

Now that you have your entire cluster instrumented on Kubernetes, let's look at how you can use New Relic to its best. First, head over to the New Relic dashboard, then select your Kubernetes cluster. This will immediately give you an overview of your entire cluster, broken down to a namespace level. You can see all the pods running in each namespace, any issues they have, etc... You can use the filters at the top to drill down to specific pods and if you scroll down the page, you will see all recent events that happened in your cluster. If your cluster is small, this page alone should help you with your entire monitoring stack. However, if your cluster is larger (as it usually is), you will need to get more specialized insights.

## Kubernetes events

To start, go to the Kubernetes Events section. This is argulably the most important page in New Relic with regards to Kubernetes. Since all events are logged here, any issue you face in your cluster will also be logged here. If your pods/nodes/containers go out of memory, disk, or anything else, this page will log it. By disabling the "Warnings only" check box, you can get events for everything. You will know exactly when each container in each pod started, how long it ran, what problems it faced, and so on down to a granular level. This is going to be a major help when figuring out what went wrong with something.

The next most powerful debugging tool is using NRQL queries. Let's look at a few common ones:

## NRQL queries

To get further events and information from NRQL, you can use this query:

```
SELECT event.involvedObject.name, event.message FROM InfrastructureEvent WHERE `event.involvedObject.name` = '<pod-name>' AND (clusterName = 'prod' OR `k8s.cluster.name` = 'prod') SINCE 24 HOURS AGO 
```

Note that you can change the object to be a node name or container name and this query will still work. For example:

```
SELECT event.involvedObject.name, event.message FROM InfrastructureEvent WHERE `event.involvedObject.name` = 'ip-192-168-162-138.ec2.internal' AND (clusterName = 'prod' OR `k8s.cluster.name` = 'prod') SINCE 24 HOURS AGO 
```

would work as well. Another thing you may experience with running Kubernetes pods is that there might be memory issues that arise from time to time. To observe this, you can use the below query:


```
SELECT max(memoryUsedBytes), max(memoryLimitBytes)
FROM K8sContainerSample 
WHERE podName = '<pod>' 
  AND containerName = '<container>' 
  AND clusterName = '<cluster>'
SINCE 1 day ago TIMESERIES
```

This will give you a graph that shows the memory used vs the memory limit given to the pod. If the memory used reaches the memory limit, you can expect OOM issues (not always however). You can then use the pods shown by the graph in the event explorer to get specific about what happened. You can also use the APM page to see if there was a memory leak from the application or if garbage collection is not happening prpoerly, etc...

You can also look at Kubernetes Jobs from here. Since jobs start and stop (shutdown), you might have no insights as to what happened to a job pod even if you have the logs. With events, you can get an idea as to the happening of a single job. However, if you wanted to get statistics about your job, you will need NRQL. For example, let's say you wanted to get the longest running job in the month of May. You would use this query:

```
SELECT max(completedAt - createdAt) AS 'Longest Duration (seconds)'
FROM K8sJobSample
WHERE completedAt IS NOT NULL
  AND clusterName = 'cluster'
  AND namespaceName = 'namespace'
  AND jobName LIKE 'your-job-%'
FACET jobName
SINCE '2025-05-01' UNTIL '2025-06-01'
LIMIT 10
```

This would give a table with the top 10 longest running jobs in May. You can then change the month limit to figure out the trend of your longest running jobs so that you can adjust them either from the application side or the infrastructure side. This isn't remotely close to what NRQL can offer you. To get a full list of metrics available to you, use:

```
SELECT keyset() FROM K8sJobSample SINCE 1 day AGO
```

This will output a JSON list of all the metrics and options that Cron jobs push to New Relic. This same approach can be used for other items such as `K8sContainerSample`, `K8sPodSample`, `K8sNodeSample`, etc...

Addtionally, you can use the New Relic AI to ask questions in plain english and have them transalted to NRQL queries, which will be automatically run to give you a direct answer. Note that if you were to use other generative AI such as chatgpt, you would get mostly correct NRQL queries, but the keysets that they use might be incorrect on occasion. For example, it might give you a query which uses `endTime` instead of `completedAt`, and New Relic doesn't do a great job of letting you know when you are trying to use keys that don't exist. Instead it gives you a blank response and you are left wondering if you don't have data. So if you notice some issue like that, you can use the `SELECT keyset()` to get the keysets first, then feed the result into gen AI so their answers can be more accurate.