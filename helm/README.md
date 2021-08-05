# Lyft Helm Chart

I just stole the repo and made a couple helm charts for easy installing.

In this directory there are two charts, flinkk8soperator and wordcount example. Flinkk8soperator is the helm chart for this repos operator. It is currently installed into the `wk-dev-aws` cluster in the `flink-lyft` namespace. The second chart is the WordCount example which is deployed to the same namespace.

I put it into a namespace so that we can purge it at the end, but also it makes it easy to view the resources. For example:

``` sh
kubectl get all -n flink-lyft
```

``` stdout
NAME                                                          READY   STATUS    RESTARTS   AGE
pod/flinkoperator-5db47d9995-44cvz                            1/1     Running   0          81m
pod/wordcount-operator-example-ed287ce0-jm-8676659796-qgjnl   1/1     Running   0          4m5s
pod/wordcount-operator-example-ed287ce0-tm-5b7ff66f-drbsj     1/1     Running   0          4m5s
pod/wordcount-operator-example-ed287ce0-tm-5b7ff66f-mlsvk     1/1     Running   0          4m5s

NAME                                          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                         AGE
service/wordcount-operator-example            ClusterIP   172.20.59.70    <none>        6123/TCP,6125/TCP,6124/TCP,8081/TCP,50101/TCP   4m6s
service/wordcount-operator-example-ed287ce0   ClusterIP   172.20.149.55   <none>        6123/TCP,6125/TCP,6124/TCP,8081/TCP,50101/TCP   4m6s

NAME                                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/flinkoperator                            1/1     1            1           81m
deployment.apps/wordcount-operator-example-ed287ce0-jm   1/1     1            1           4m6s
deployment.apps/wordcount-operator-example-ed287ce0-tm   2/2     2            2           4m6s

NAME                                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/flinkoperator-5db47d9995                            1         1         1       81m
replicaset.apps/wordcount-operator-example-ed287ce0-jm-8676659796   1         1         1       4m6s
replicaset.apps/wordcount-operator-example-ed287ce0-tm-5b7ff66f     2         2         2       4m5s
```

As you can see there are 4 pods running currently, `pod/flinkoperator-****` is owned by `deployment.apps/flinkoperator` and is responsible for creating new flink "clusters". There is one cluster per job. The next pod is `pod/wordcount-operator-example-******-jm-******` and it is the job manager and is associated with `deployment.apps/wordcount-operator-example-******-jm` while  pod/wordcount-operator-example-******-tm-******` is the task manager and is associated with `deployment.apps/wordcount-operator-example-******-tm`.

By looking at the taskmanager log outputs you can see that 3 "slots" were used while processing the file. Because the slot that processed the request logs with a prefix of the slot number. [Here](https://splunk.workiva.net/en-US/app/search/search?q=search%20index%3Dkinesis-wk-dev-aws%20%22pod.name%22%3D%22wordcount-operator-example-*-tm-*%22%20message%3D%22*%3E%20\(*%22&display.page.search.mode=smart&dispatch.sample_ratio=1&earliest=1628190014&latest=1628190974&sid=1628191257.23575_E1E254A8-D1AC-4B03-A4B7-AB2DD555FC19) is the splunk query. In this data you can see the sources came from two containers.

If you look at the job manager logs you can see that the task was shared into 3 "Flat Maps". [Here](https://splunk.workiva.net/en-US/app/search/search?q=search%20index%3Dkinesis-wk-dev-aws%20%22pod.name%22%3D%22wordcount-operator-example-*-jm-*%22&display.page.search.mode=smart&dispatch.sample_ratio=1&earliest=1628190014&latest=1628190974&sid=1628191291.23577_E1E254A8-D1AC-4B03-A4B7-AB2DD555FC19) is the splunk query for the corresponding job manager logs.

The services listed allow for the deployment to be reached by k8s-internal services, currently the operator is deployed with a dummy ingress but I was unable to get that working using `kubectl proxy`.

Finally [here](https://github.com/lyft/flinkk8soperator/blob/master/docs/user_guide.md) is a user guide for using this operator.

If you want to ensure the job gets ran again, go ahead and delete the word count example and install it by running.

``` sh
helm delete wordcount-example -n flink-lyft
helm upgrade --install wordcount-example ./helm/wordcount-example -n flink-lyft
```

likewise for the operator itself should you need to delete it....

``` sh
helm delete flinkk8soperator -n flink-lyft
helm upgrade --install flinkk8soperator ./helm/flinkk8soperator -n flink-lyft
```
