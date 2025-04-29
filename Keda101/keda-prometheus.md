# Using KEDA with prometheus

Scaling your application based on the number of requests it receives is a common situation. If you wanted to do that with KEDA, you could use the [KEDA http add-on](https://github.com/kedacore/http-add-on). This solution is great if your application is small and you don't mind the architecture of your existing application changing. However, if you want to scale based on requests without going through all that, you aren't left with many options. To begin with, you need a tool that can accurately capture the number of requests coming into each application. Service meshes have this information, but no service mesh out there handles pod scaling since that is well out of their scope. So you have KEDA which doesn't have access to metrics such as request counts, and service meshes, which have access to that information, but aren't able to scale your pods. How do you mix the two together? The answer is Prometheus.

Using KEDA with Prometheus can allow you to scale your application based on metrics that you would otherwise not have access to. Prometheus supports a wide variety of metrics out of the box, and you can use it in conjunction with other tools that push metrics into prometheus. It is also pretty simple for you to push your own metrics from your applications to Prometheus, and KEDA having a scaler based on Prometheus means you can scale based on just about anything. If you need to get a refresher on Prometheus as well as some idea about PromQL, head over to the [Prometheus](../201/monitoring/prometheus.md) section. In this case, we will be using a mix of Prometheus and [LinkerD](../ServiceMesh101/what-is-linkerd.md) to scale your KEDA application based on the number of requests it receives. We will be using the version of Prometheus that comes bundled with LinkerD, so you don't have to set up Prometheus in your cluster separately.

## Concept

When LinkerD is installed, it will deploy a sidecar proxy container to run alongside your application container and intercept any request information your application receives. It will then push these metrics to Prometheus every 10 seconds. You can find a list of all metrics LinkerD pushes [here](https://linkerd.io/2.15/reference/proxy-metrics/). We will then implement the KEDA Prometheus plugin to poll Prometheus every 10 seconds, and scale based on the results of polling.

## Requirements

You will of course need a Kubernetes cluster. A cluster such as [Minikube](https://minikube.sigs.k8s.io/docs/start/) will do fine. You then need to set up LinkerD on the cluster, which is a pretty uncomplicated task. Just follow the [doc](https://linkerd.io/2.15/tasks/install/). Make sure you also install [linkerd-viz](https://linkerd.io/2.15/reference/cli/viz/) following the install instructions so that Prometheus gets set up and automatically configured. Once all this is done, start up a simple application like a nginx server:

```
kubectl run nginx-pod --image=nginx --restart=Never --port=80
```

Expose the port:

```
kubectl expose pod nginx-pod --type=NodePort --port=80 --name=nginx-service
```

Check the services from the below command and use the IP of the server to access the nginx server:

```
kubectl get svc
```

Now that nginx is up, let's inject LinkerD to act as a proxy to this server:

```
kubectl get deploy -o yaml | linkerd inject - | kubectl apply -f -
```

The above command will add Linkerd to all pods in the default namespace. If you run a `kubectl describe` you should be able to see there is a Linkerd proxy container in addition to the Nginx container running in your proxy pod. If there isn't restart the pod and it should start up with the linkerd proxy pod injected. Before we add keda into the mix, let's take a look at how we will be polling prometheus. For starters, let's take a look at the Prometheus dashboard. To get access to this, run:

```
kubectl get po -n linkerd-viz
```

Take the full name of the prometheus pod, then port forward it:

```
kubectl port-forward <prometheus-pod-name> 9090:9090
```

You should now be able to open up localhost:9090 in your browser and get access to your pod. Let's run a PromQL query to check the request count. Linkerd sends a custom metric called `request_total` which can be used here. For a complete list of all custom metrics linkerd makes available, check the [official documentation](https://linkerd.io/2.15/reference/proxy-metrics/). This is the promQL we will use:

```
sum(rate(request_total{app="nginx",job="linkerd-proxy"}[3m])) by (app)
```

Let's break down this query. First, we use `request_total`, which is the total sum of all requests the nginx pod receives. This is a number that keeps increasing forever. Every request that comes into your pod gets appended to this count. So if you have 100 requests coming in this hour and another 100 in the next, if you were to check this metric in the next hour, it would show 200. This metric has specifications such as `app` and `job` which are supposed to filter down the number of requests so that you only count the requests in the pods you want, and the job is set to the `linkerd-proxy` container since that is what is collecting all the metrics. Next to that is `rate` which is essentially used to calculate the difference in the metric (rate of change). This rate shows how the number of requests varies in the specific period (3 minutes). So if you were to extend this to an hour and go back to our previous example, while the number of requests in the second hour would total 200, the rate would show 100, because 100 of the requests came in hour 1 and the other 100 came in hour 2. So the rate of change within 1 hour would be 100. Finally, you have the `sum` function. If you look at your cluster now, you are only running 1 nginx pod. Therefore, the `sum` function currently does nothing. However, when your pod starts to scale and the number of your pods increases, the traffic coming into your pods will also split. At this point, getting the total number of requests per a single pod isn't going to help anything because the total number of requests is the requests handled by all the pods, which is where `sum` comes in and totals the request count.

If you head over to the Prometheus dashboard, you should be able to use the above query. Switch to the graph tab to get a better visualization of the requests and try manually reloading the nginx page. The graph should show an increase in the request counts.

## Setting up KEDA

Now that everything is set up from the linkerd/Prometheus side, let's take a look at the KEDA side of things. We will be using the in-built [prometheus scaler](https://keda.sh/docs/2.14/scalers/prometheus/) for the scaling, and this is the yaml of the scaled object:

```
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: nginx-request-scaler
spec:
  scaleTargetRef:
    name: nginx 
  pollingInterval: 10 # Seconds
  cooldownPeriod: 60 
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.linkerd-viz.svc.cluster.local:9090
      metricName: request_total
      threshold: "5.0"
      query: sum(rate(request_total{app="nginx",job="linkerd-proxy"}[3m])) by (app)
```

Let's take a closer look at the yaml above. The resource is of kind ScaledObject, which is named `nginx-request-scaler`, and it scales the deployment called `nginx`. If you have named your nginx deployment something else, change the `scaleTargetRef` to that. Below that comes the threshold for polling and cooling. In this case, we ask keda to poll Prometheus every 10 seconds and start scaling immediately if the request rate is over the threshold. We also tell it to start scaling down 60 seconds after the request rate has dropped below the threshold. Note that keda won't immediately start terminating replicas just because the cooldown period has been reached. This is because in a real production environment loads can fluctuate, so having replicas scale down quickly is not ideal. You also have the variables `minReplicaCount` and `maxReplicaCount`, which is similar to how HPAs have their limits defined.

The next block defines the triggers. In this case, we have specified the use of the `prometheus` plugin, and asked keda to poll `http://prometheus.linkerd-viz.svc.cluster.local:9090`, which is the endpoint for Prometheus if you installed it via Linkerd. The metric we will be using is called `request_total`, which is a metric you can read about in more detail in the [official documentation](https://linkerd.io/2.15/reference/proxy-metrics/). The query is the one we observed before in the Prometheus dashboard, and the threshold is set to 5. This means if the number of requests per second goes beyond 5, a new replica will spin up. If it gets to 10, another starts up, and so on.

Assuming you already have keda installed in your cluster, all you have left to do is to deploy this scaler into your cluster like you would any other manifest file. Make sure to deploy it in the same namespace as your nginx pod.

## Testing

You are now ready to see the scaling in action. To generate a minor load so we can test the scaling, we will be using a simple tool like [Apache Bench](https://httpd.apache.org/docs/2.4/programs/ab.html) which is included in the ```apache2-utils``` package. Use this to ping your nginx server:

```
ab -n 100 -c 10 http://localhost/
```

This should send 100 requests to your Nginx application. Once you have run this command, go to the Prometheus dashboard, adjust the time, and you should be able to see the request count going up. Within 30 seconds, your replicaset should have started scaling, so running `kubectl get po` should reveal multiple replicas of the nginx pod starting up. Since you have linkerd-viz installed, you can also see how the requests are getting load balanced across the pods using the linkerd dashboard.

## Considerations

Now that you have seen how KEDA and Prometheus work together with linkerd to scale your application workloads, you might have already noticed a few considerations you need to take into account. Both Linkerd and KEDA are well-used tools that are generally used in production workloads, so there is little reason to be worried about them breaking. While the same holds true for Prometheus, you might notice here that Prometheus runs in a single pod. For starters, this Prometheus instance is introduced by Linkerd but is not supposed to be an official Prometheus instance that is supposed to cater to your entire cluster. If you need something like that, you will have to set up Prometheus yourself and bring the linkerd metrics to your Prometheus instance. [This document](https://linkerd.io/2.15/tasks/external-prometheus/) should be able to help you with that. This small prometheus instance only runs one replica, but if you are going to be relying on it for your scaling in a production environment, it is advisable to extend it to run several replicas across multiple availability zones. It also helps to have alarms and alerts configured so that you know that the Prometheus instance is running fine. Next, while keda and karpenter are robust applications, if you are relying on them to keep your production workloads running, it's best to have them monitored too since they are moving parts in your infrastructure.

## Conclusion

This brings us to the end of this section on using Keda with Prometheus to scale your pods based on the number of replicas. If you want to try out other scalers, make sure you check out the [official samples page](https://github.com/kedacore/samples). Now that we have a good idea of how Prometheus can be used to do request based scaling, let's take a look at how we can leverage this scaling effectively with proper scaling profiles.

[Next: Scaling profiles](./scaling-profiles.md)