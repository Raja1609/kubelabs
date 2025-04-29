# Autoscaling helpers

We have now discussed HPA's, VPA's, and you might have even read the section on [KEDA](../Keda101/what-is-keda.md) to learn about advanced scaling. Now that you know all there is to about scaling, let's take a step back and look at a few important things to consider when it comes to scaling. In this section, we will discuss:

- Readiness/liveness/startup probes
- Graceful shutdowns
- Annotations that help with scaling.
- Pod priority/disruption
- Pod requests/limits
- Rollover strategy
- Pod topology skews

You may have already come across these concepts before, and just about every Kubernetes-based tool uses them to ensure stability. We will discuss each of the above points and follow up with a lab where we test out the above concepts using a simple Nginx server.

# Probes

What are readiness/liveness/startup probes and why are they useful for autoscaling? Let's break down each type of probe.

- Readiness probe: As the name suggests, this probe checks to ensure that your container is ready.

In order to do this, you could implement several methods. The simplest and most frequently used is the http get method. You simply point the readiness probe at your containers' endpoint, then have the probe ping it. If the response to the ping is 200 OK, your pod is ready to start receiving traffic. This is incredibly useful since it's rare that your application is ready to go immediately after starting up. Usually, the application needs to establish connections with databases, contact other microservices to get some starting information, or even run entire startup scripts to prepare the application. So it may take a couple of seconds to a couple of minutes for your application to be ready to take traffic. If any requests come in within this period, they will be dropped. With the readiness probe, you can be assured that this won't happen.

Apart from a simple HTTP get requests, you could also run TCP commands to see if ports are up, or even run a whole bash script that executes all commands to determine whether your pod is ready. However, this probe only continually checks to see if your app is ready to take requests. It blocks off any traffic if it starts to notice that the probe is failing. If you only have a readiness probe in place, even if your app has gone into an error state, Kubernetes will only prevent traffic from entering that pod until the probe starts to pass. It will not restart the failed application for you. This is where liveness probes come in.

- Liveness probe: Check if your application is alive.

A liveness and readiness probe do almost the same thing, except a liveness probe restarts the pod if it starts failing, unlike the readiness probe which only stops traffic to the pod until the probe starts succeeding. This means that the liveness probe should come after the readiness probe. You could say something like: if my container's port 8080 isn't being reached, stop sending traffic (readiness probe). If it is still unreachable after 1 minute, fail the liveness probe and restart the pod since the container has likely crashed, gone OOM, or is meeting some other pod or node constraints.

- Startup probe: A probe similar to the other two, but only runs on startup.

If your pod takes a while to initialize, it's best to use startup probes. Startup probes ensure that your pod started correctly. You can even use the same endpoint as the liveness probe but with a less strict wait time. When your pod is already running, you don't expect the endpoint to go down for more than a few seconds if at all. However, when starting up, you can expect it to be down until your application finishes initializing. This is why there is a separate startup probe instead of re-using the existing liveness probe.

So how do these probes help with autoscaling? In the case where replicas of pods increase and decrease meaning that instances of your application are provisioned and de-provisioned, you need to make sure there is no downtime. This is where all the above probes come into play. When the load into your application increases and replicas of your pods show up, you don't want any traffic served until they are ready. If they have issues getting prepared and don't start after a while, you want them to restart and try to auto-recover. Finally, if a pod fails after running for a while, you want traffic to be blocked off and that pod restarted. This is why these probes are necessary for autoscaling.

## Graceful shutdowns

Now let's take a look at graceful shutdowns. If you were running a website that had high traffic and your pods scaled up during high traffic, they must scale back down after a while to make sure that your infrastructure costs are kept as efficient as possible. However, if your Kubernetes configuration was to immediately kill the pod off while the traffic was being served, that might result in a few requests being dropped. This is where graceful shutdowns are needed.

Depending on the type of web application you are running, you may not need to configure graceful shutdowns from the Kubernetes configuration. Instead, the application framework itself might be able to intercept the shutdown signal Kubernetes sends and automatically prevent the application from receiving any new traffic. For example, in SpringBoot, you can enable graceful shutdowns simply by adding the config `server.shutdown=graceful` into your application config. However, if your application framework doesn't support something like this, or you prefer to keep your Kubernetes and application configurations separate, you might consider creating a `shutdown` endpoint. We will do this during the lab.

While microservices generally take in traffic through their endpoints, your application might differ. Your application might do batch processing by reading messages off RabbitMQ, or it might occasionally read a database and transform the data within it. In cases like this, having the pod or job terminated for scaling considerations might leave your database table in an unstable state, or it might mean that the message your pod was processing never ends up finishing. In any of these cases, graceful shutdowns can keep the pod from terminating long enough for your pod to either finish what it started or ensure a different pod can pick up where it left off.

One major problem you might encounter if you are using a command string to start your application is if Kubernetes doesn't recognize your command string as a primary process and fails to propagate the sigterm command to your application. This can happen if you have multiple commands chained together before your main command runs, meaning Kubernetes is unable to determine which command is the main one. To explicitly define a command as a main command, use `exec`. For example:

```
cp somefile . && rm -rf somefolder && exec java -jar yourapplication.jar
```

Without the `exec`, the sigterm will be sent to the first `cp` command and will not affect your actual `java` command.

If the jobs you are running are mission-critical, and each of your jobs must run to completion, then even graceful shutdowns might not be enough. In this case, you can turn to annotations to help you out.

## Annotations

Annotations are a very powerful tool that you can use in your Kubernetes environments to fine-tune various aspects of how Kubernetes works. If, as mentioned above, you need to make sure that your critical job  runs to completion regardless of the node cost, then you might want to make sure that the node that is running your job does not de-provision while the job is still running on it. You can do this by adding the below annotation:

```
annotations:
 "cluster-autoscaler.kubernetes.io/safe-to-evict": "false"
```

This will ensure that your node stays up even if there is only 1 job running on it. This will certainly increase the cost of your infrastructure since normally, Kubernetes would relocate jobs and de-provision nodes to increase resource efficiency. It will only shut down the node once no jobs are running that have this annotation left. However, if you don't want the nodes to shut at all, you can add a different annotation that ensures that your nodes never scale down:

```
cluster-autoscaler.kubernetes.io/scale-down-disabled
```

This annotation should be applied directly to a node like so:

```
kubectl annotate node my-node cluster-autoscaler.kubernetes.io/scale-down-disabled=true
```

Obviously, this is not a recommended option unless you have no other choice regarding the severity of your application. Ideally, your jobs should be able to handle shutdowns gracefully, and any jobs that start in place of the old ones should be able to complete what the previous job was doing.

Another annotation that helps with autoscaling is the `autoscaling.alpha.kubernetes.io/metrics` annotation which allows you to specify custom metrics for autoscaling, like so:

```yaml
metadata:
  annotations:
    autoscaling.alpha.kubernetes.io/metrics: '[{"type": "Resource", "resource": {"name": "cpu", "targetAverageUtilization": 80}}]'
```

Now that we have looked at annotations, let's look at how pod priority and disruptions budgets can help you with scaling.

**Pod Priority**:

You can influence the Cluster Autoscaler by specifying pod priority, and ensuring critical pods get scheduled first.

 ```yaml
  spec:
    priorityClassName: high-priority
 ```

If you have an application that handles all incoming traffic and then routes it to a second application, you would want the pods of the external-facing application that handles traffic to have more priority when scheduling. If you have jobs that run batch workloads, they might take lesser priority compared to pods that handle your active users.

Earlier, we discussed using annotations to prevent disruptions due to scaling. However, those methods were somewhat extreme, making the node stay up even if 1 pod was running or making sure the node never went down at all. What if we wanted to allow scaling but also wanted to maintain some control over how much this scaling was allowed to disrupt our workloads? This is where pod disruption budgets come into play.

**Pod Disruption Budget (PDB)**:

A Pod Disruption Budget (PDB) is a Kubernetes resource that ensures a minimum number of pods are always available during voluntary disruptions, such as maintenance or cluster upgrades. It prevents too many pods of a critical application from being taken down simultaneously, thus maintaining the application's availability and reliability.

Let's say you have a Kubernetes Deployment with 5 replicas of a critical web service. You want to ensure that at least 3 replicas are always available during maintenance activities. You can create a PDB with the following YAML configuration:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-service-pdb
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: web-service
```

This is not very different to how other Kubernetes resources work where the external resource applies its configuration by selecting the deployment with a label. The components of the above PodDisruptionBudget are as follows:

- `apiVersion: policy/v1`: Specifies the API version.
- `kind: PodDisruptionBudget`: Indicates that this resource is a PDB.
- `name: web-service-pdb`: The name of the PDB.
- `minAvailable: 3`: Specifies that at least 3 pods must be available at all times.
- `selector`: Defines the set of pods the PDB applies to. In this case, it matches pods with the label `app: web-service`.

### How it Works

1. **Normal Operation**:
   - Under normal conditions, all 5 replicas of the web service are running.

2. **During Disruption**:
   - When a voluntary disruption occurs (e.g., node maintenance or a manual pod eviction), the PDB ensures that at least 3 out of the 5 pods remain running.
   - If an attempt is made to evict more than 2 pods at the same time, the eviction will be blocked until the number of available pods is at least 3.

Now that we're clear on disruption budgets, let's look at node affinities.

**Node Affinity**:

Node affinity rules influence where pods are scheduled, indirectly affecting autoscaling decisions.

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
```

Ensuring that nodes & pods are started in different zones will ensure high availability when a zone goes down. This also brings us to an important point if you run a large application with many microservices. Each replica of each microservice requires its own IP, and in a normal subnet, you only have 250 of them. Considering that each node you bring up has several daemonsets running on them that reserve their own IPs, coupled with each microservice replica needing its own IP, you might quickly find yourself in a position where you have run out of IPs and the pod is unable to start because the CNI doesn't have any IPs left to assign. In this case, having several subnets spread evenly across several availability zones is the answer. But even then, it is possible that the cluster autoscaler (or Karpenter if you use that instead), will end up provisioning nodes in a subnet that is about to run out of IPs. So having zonal topology constraints at a pod level will ensure that the pods are spread out and demand that nodes be spread across the subnets, thereby reducing the chance of IP address exhaustion.

### Additional autoscaling considerations

This is the very start of looking into problems you could run into while scaling. Depending on how much you scale and what you scale with, you might run into unpredictable issues. If you were to take an application designed to run on a static machine, and then scale it as-is, the problems would become apparent. So if you are planning to scale your production workloads, make sure you have proper monitoring and logging in place. For more on this, take a look at how you can [run filebeat as a sidecar](../Logging101/filebeat-sidecar.md) so that even if you were to scale your applications to hundreds of replicas, you would still have perfect logging over each of them. This is crucial because at one point, sifting through log files is no longer an option. There would be so many that you would probably have trouble finding one among the thousands. Monitoring is also pretty important. Even with graceful shutdowns enabled, you might sometimes see requests getting dropped. When this happens, you need to know why the request was declined. Perhaps the node went out of memory, or the pod unexpectedly crashed. Maybe there was some third party that intervened with the graceful shutdown. So having something that can scrape the logs and Kubernetes events is pretty useful so can go back and see what failed. Of course, proper logging and monitoring is crucial in any production environment. However, when it comes to Kubernetes where nodes, pods, and other resources come up and go down regularly, you wouldn't have a trace of what happened to the resource when you wanted to debug something.

You also have to consider that the tool used to perform scaling might run into issues. For example, if you were to use Karpenter for node scaling and all Karpenter pods went down, node scaling would no longer happen. This means that pods might come up and remain in a pending state since there aren't enough nodes to run the pods. To counter this, you need to run multiple replicas of Karpenter and properly set the node affinity rules so that there is never a situation where Karpenter goes down completely. Additionally, tools like KEDA which are used for pod autoscaling ensure low downtime by running reserve pods that can come up if the main pod goes down. Other autoscaling tools like the cluster autoscaler and HPA/VPA are already built with resilience in mind, so you don't have to worry too much about it. However, if you were to something like scale your pods based on Prometheus metrics, then it is your responsibility to make sure that Prometheus is running with high availability perhaps with the help of Thanos.

## Lab

Now, let's get started on the lab and take a practical look at all the things we discussed above. For this, it's best to use a cloud provider for your Kubernetes cluster as opposed to Minikube, since we need to have multiple nodes so we can take a look at node scaling. Even a multi-node cluster that you run on your local machine is fine. For this, we will be using the Nginx image as the application and come up with our own Nginx deployment yaml that incorporates most of the attributes discussed above.

### Step 1: Create a Kubernetes Deployment Manifest

Create a deployment manifest file named `nginx-deployment.yaml` with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /readiness
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        startupProbe:
          httpGet:
            path: /startup
            port: 80
          initialDelaySeconds: 0
          periodSeconds: 10
```

Here we see all the different types of probes we previously discussed. We first have a `livenessProbe` that checks whether our pod is alive every 10 seconds, a `readinessProbe` that checks if the pod is ready to serve traffic every 5 seconds, and a startup probe that checks to see if the pod has started up properly.Now that we have defined the various probes, we need to define the actual endpoints. Otherwise, the probes will ping these missing endpoints and our application will never start.

### Step 2: Create a ConfigMap for Custom NGINX Configuration

Create a ConfigMap file named `nginx-configmap.yaml` with the following content to define custom health check endpoints:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  default.conf: |
    server {
        listen 80;
        
        location /healthz {
            access_log off;
            return 200 'OK';
            add_header Content-Type text/plain;
        }

        location /readiness {
            access_log off;
            return 200 'OK';
            add_header Content-Type text/plain;
        }

        location /startup {
            access_log off;
            return 200 'OK';
            add_header Content-Type text/plain;
        }

        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }
    }
```

### Step 3: Apply the ConfigMap

```sh
kubectl apply -f nginx-configmap.yaml
```

### Step 4: Create a Kubernetes ConfigMap Volume Mount in Deployment

Update the `nginx-deployment.yaml` to mount the ConfigMap as a volume:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config-volume
          mountPath: /etc/nginx/conf.d
          subPath: default.conf
        livenessProbe:
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /readiness
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        startupProbe:
          httpGet:
            path: /startup
            port: 80
          initialDelaySeconds: 0
          periodSeconds: 10
      volumes:
      - name: nginx-config-volume
        configMap:
          name: nginx-config
```

### Step 5: Apply the Updated Deployment Manifest

```sh
kubectl apply -f nginx-deployment.yaml
```

### Step 6: Verify the Deployment

To check the status of your deployment and the probes, you can use the following commands:

```sh
# Check the status of the deployment
kubectl get deployments

# Check the status of the pods
kubectl get pods

# Describe a pod to see probe details
kubectl describe pod <pod-name>
```

With this configuration, you should have a robust deployment of NGINX with proper health checks using readiness, liveness, and startup probes.

Now, let's move on to graceful shutdowns.

### Step 1: Define a PreStop Hook in Your Deployment Manifest

Update the deployment manifest (`nginx-deployment.yaml`) to include a `preStop` hook that will handle the graceful shutdown:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config-volume
          mountPath: /etc/nginx/conf.d
          subPath: default.conf
        livenessProbe:
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /readiness
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        startupProbe:
          httpGet:
            path: /startup
            port: 80
          initialDelaySeconds: 0
          periodSeconds: 10
        lifecycle:
          postStart:
            exec:
              command: ["/bin/sh", "-c", "echo 'nginx started'"]
          preStop:
            exec:
              command: ["/bin/sh", "-c", "nginx -s quit && sleep 30"]
      volumes:
      - name: nginx-config-volume
        configMap:
          name: nginx-config
```

### Step 2: Apply the Updated Deployment Manifest

Apply the updated deployment manifest to your Kubernetes cluster:

```sh
kubectl apply -f nginx-deployment.yaml
```

This introduces two new attributes:

1. **terminationGracePeriodSeconds:** This sets the period (in seconds) that Kubernetes will wait after sending a SIGTERM signal to the container before forcefully terminating it with a SIGKILL signal. The default value is 30 seconds, but it can be adjusted based on your application's requirements. In this example, it is set to 60 seconds.

2. **preStop Hook:** This lifecycle hook executes a command just before the container is terminated. In this case, the command is `nginx -s quit && sleep 30`.
    - `nginx -s quit`: This command gracefully stops the NGINX process, allowing it to finish serving ongoing requests.
    - `sleep 30`: This ensures that the container waits for 30 seconds before fully shutting down. This additional sleep period provides extra time for ongoing requests to complete and for NGINX to shut down cleanly.

### Step 3: Verify the Graceful Shutdown

To verify that the graceful shutdown is working correctly, you can simulate a termination of one of the NGINX pods and observe the logs and behavior:

1. **Delete an NGINX Pod:**

    ```sh
    kubectl delete pod <nginx-pod-name>
    ```

2. **Observe Logs and Behavior:**

    Use the following command to check the logs of the NGINX pod:

    ```sh
    kubectl logs <nginx-pod-name> --previous
    ```

    When observing the logs from the nginx pods, you should be able to see that a graceful shutdown is being performed.

With that, we cover graceful shutdowns. We will be skipping annotations for this lab since the annotations themselves are uncomplicated and applying them to your deployments or nodes is very straightforward. So let's jump right ahead to pod priority.

To introduce Pod Priority to our Nginx Deployment, we need to define a `PriorityClass` and then reference it in our Deployment's Pod spec.

### Step 1: Define a PriorityClass

First, you need to create a `PriorityClass` resource. This resource defines a priority value and an optional description.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
globalDefault: false
description: "This priority class is used for high priority pods."
```

### Step 2: Reference the PriorityClass in the Deployment

Next, update the Deployment to use the newly defined `PriorityClass`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      priorityClassName: high-priority
      terminationGracePeriodSeconds: 60
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config-volume
          mountPath: /etc/nginx/conf.d
          subPath: default.conf
        livenessProbe:
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /readiness
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        startupProbe:
          httpGet:
            path: /startup
            port: 80
          initialDelaySeconds: 0
          periodSeconds: 10
        lifecycle:
          postStart:
            exec:
              command: ["/bin/sh", "-c", "echo 'nginx started'"]
          preStop:
            exec:
              command: ["/bin/sh", "-c", "nginx -s quit && sleep 30"]
      volumes:
      - name: nginx-config-volume
        configMap:
          name: nginx-config
```

### Explanation

1. **PriorityClass Resource**:
    - `name`: The name of the priority class (`high-priority` in this example).
    - `value`: The priority value assigned to this class. Higher values indicate higher priority.
    - `globalDefault`: Indicates if this should be the default priority class for Pods that do not specify any priority class.
    - `description`: A human-readable description of the priority class.

2. **Deployment Update**:
    - `priorityClassName`: Added to the Pod spec to assign the priority class to the Pods created by this Deployment.

By adding the `PriorityClass` and referencing it in your Deployment, you ensure that the Pods in this Deployment are given a higher priority during scheduling and eviction processes compared to other Pods with lower priority or no specified priority class. Lower priority in this case would be a priority less that 10000 (which is what we have defined as high).

Finally, let's get to PodDisruptionBudgets. To add a Pod Disruption Budget (PDB) to your deployment, you need to create a PDB resource. A PDB ensures that a certain number of pods in a deployment are available even during voluntary disruptions (such as draining a node for maintenance). Below is our updated configuration with a PDB added. The deployment file remains essentially the same. The disruption budget comes from a new manifest of kind `PodDisruptionBudget`:

### Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      priorityClassName: high-priority
      terminationGracePeriodSeconds: 60
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-config-volume
          mountPath: /etc/nginx/conf.d
          subPath: default.conf
        livenessProbe:
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /readiness
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        startupProbe:
          httpGet:
            path: /startup
            port: 80
          initialDelaySeconds: 0
          periodSeconds: 10
        lifecycle:
          postStart:
            exec:
              command: ["/bin/sh", "-c", "echo 'nginx started'"]
          preStop:
            exec:
              command: ["/bin/sh", "-c", "nginx -s quit && sleep 30"]
      volumes:
      - name: nginx-config-volume
        configMap:
          name: nginx-config
```

### Pod Disruption Budget YAML

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
  labels:
    app: nginx
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: nginx
```

### Explanation:
- **minAvailable: 2**: This specifies that at least 2 pods must be available at all times.
- **selector**: Ensures that the PDB applies to the pods matching the specified labels (`app: nginx`).

Apply the configuration:

```sh
kubectl apply -f nginx-pdb.yaml
```

Now your number of pods won't go below the minimum available pod count meaning that if pods are evicted due to autoscaling, if a new version is deployed, or if your pods are supposed to restart for any reason, at least 2 pods will always be up. This, however, will not consider a case where your pod or node goes out of memory, becomes unreachable, or unschedulable. If your node doesn't have enough resources to give, even a PDB insisting that the pod needs to stay up doesn't work. The same applies if the node suddenly were to get removed. To minimize the change of this happening, you will properly have to set pod requests and limits so that the resource requirements of a pod never exceed what the node can provide. On that note, let's take a look at pod requests and limits.

## Pod requests/limits

You have undoubtedly seen pod requests and limitss and likely even used them at some point. Requests/limits define the minimum and maximum CPU & memory a pod is allowed to take. To break it down:

- Requests: A pod will not schedule on a node unless the pods' memory and CPU requets are fulfilled. If there are no nodes that can fulfill a pods requested resources, a new node will have to be added. If you use cluster autoscaler or Karpenter, this resource requirement will be noted and machines will be automatically provisioned.

- Limits: This is the maximum memory a pod can consume. Once it reaches this memory limit, the pod will be terminated to prevent breaching this memory limit.

With this in mind, it might commonly make sense to put requests to a lower value and limits to a higher value. This works fine for development workflows and is called a burstable workload. However, this method has one major downside: node OOM issues. If you have set a pod to have requests of 500MB and limits of 1GB, your pod may get scheduled on a node with 700MB remaining. However, since your pod can grow in memory up to 1GB, it will continue to do so until it hits 700 MB. At this point, the node will go into a memory pressure state since it has no more memory to give. This will cause issues for all pods inside the node. The best case here is that the node evicts a pod at random so that the node goes out of the memory pressure state. The worst case is that the node itself crashes. Generally, this will result in a new node being started, but you will experience performance degradation until that happens. This might be acceptable in a development situation but quite unacceptable in production.

This is where guaranteed resource allocation comes into play. In this scenario, you would set the resources equal to the limits. So now, when a pod is looking to schedule, the scheduler will pick a machine that has the requested amount of resources in it. However, that pod will now not go over the resource limit, meaning that the node itself will never run out of memory. This means no change in node failures or random pods getting forcibly evicted.

If a pod itself reaches its memory limit, then the pod will be rescheduled. If you have properly set graceful shutdown and termination grace periods, these will be honored. This way, none of your pods will shut down without warning and cause any ongoing transactions to fail.

Note that this only applies to memory limits. When it comes to CPU limits, a general recommendation is that you don't keep any such limits in place. The CPU is elastic and can be acquired and released as needed. Even if the pod takes the full CPU of the machine, the machine itself won't crash, and once the CPU usage drops, the available CPU will be reallocated. On the other hand, if you have a stringent CPU limit in place and the pod reaches that CPU limit, the application will slow down. This is especially true when the application is starting. During the startup, the application can consume up to 10 times the normal amount of CPU. If there is a limit in place, the application startup can end up slowing down. Due to these reasons, even for production workloads, it's advisable to not have a CPU limit.

## Rollover strategy

Now, let's talk about rollover strategies. When you perform deployments, likely because you want to deploy a new version of an application, you might have to do it in the middle of peak hours. However, you don't want your application going down or even having a performance hit, which means your deployment strategy should first create new pods that will replace the existing ones before the old pods are destroyed. Kubernetes already does this by default to a certain degree by allowing 25% of the max replicas to be created and 25% to be destroyed. However, if you only run about 2 replicas that means one of them will go down before the new one comes up, resulting in a performance degredation. So you might want to strictly specify these values yourself. You can do so with:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

The above block will specify that 1 new pod is allowed to come up in addition to the existing pods during deployment and determines that the maximum number of unavailable pods is 0. If you are using around 6 pods and you want the deployments to happen faster, you can set `maxSurge` to 3 or more, which means 3 pods will come up, 3 old ones will go down, then another 3 will come up before the last old 3 pods finally shut down. However, the downside is that the additional pods might require extra resources to run on. If this happens Kubernetes will upscale and then downscale the number of nodes as needed. But if you have node disruption budgets to prevent nodes from going down during business hours and if the budget has been reached, the extra nodes will remain.

## Pod topology skews

Next, let's look at topology skews. One of the major advantages of having a microservice architecture is high availability. To reinforce this, most cloud providers give four or more availability zones per region. Each of these availability zones operates separately from one another, meaning that if one az were to go down, the other azs would be unaffected. To use these zones effectively, you should always use 2 or more azs when scheduling your Kubernetes resources. While it is true that major cloud providers don't generally have az wide outages, this is an extra step that you can put in place to keep your business availability at a maximum.

When we talk about replicas, the idea is that if one replica were to go down, the other would serve traffic, and therefore there would not be a service outage. However, if all your replicas are in a single az and the az were to go down, you would still get a service outage. So when scheduling your pods, you want each replica to be in a separate machine, and for each machine to be in a separate az. However, note that this might not be practical from a cost perspective if you only have a few small applications. Having to use separate network gateways for each az as well as having separate machines is eventually going to cost you.

As such, a compromise between the two would be to schedule pods in a different zone if a node is available there. However, if there is no other machine, schedule on the same machine or in the same availability zone. The yaml for this would be:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: nginx
```

This will skew the deployments based on the zone, but if no machines are available to schedule in a skewed manner, the skew will be ignored. However, if you have several applications that can share resources in multiple machines, you can force the pods to be scheduled in separate zones by replacing `ScheduleAnyway` with `DoNotSchedule`. If no machines are available in separate zones, the pod will refuse to schedule. Once that happens, your node scaler will kick in to satisfy the requirement.

# Conclusion

This brings us to the end of this section, where we discuss how to use various tools provided both natively and as add-ons to improve the stability of scaling, which is an essential aspect of running high availability production applications. There are a large number of other tools within the CNCF collective that help improve this stability, so don't hesitate to research the various options to get the best fit for your production workload. For more about scaling, don't forget to check out our [KEDA](../Keda101/what-is-keda.md) and [Karpenter](../Karpenter101/what-is-karpenter.md) sections.