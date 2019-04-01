# Workload Resources and Workflows with Kubernetes

## Pods and Services

Kubernetes is a database for modeling operations resources. Those resources fall into categories like "workload," "load balancing," "configuration," "storage," "cluster," and "metadata." These exercises introduce the Pod. A pod is the smallest schedulable workload resource in Kubernetes. A pod represents a set of processes that share common elements that make them "feel" like they are running on the same host.

All of the containers in a pod are configured such that they can interact like processes on a common machine. They share NET, IPC, UTS namespaces as well as a single loopback and network interface. Containers in a pod can all communicate with each other on localhost, and they compete for the same port ranges on the Ethernet interface. So, you won't be able to have two containers listen on port 80 within the same pod even if they are in different containers.

Pod resources specify discrete workloads, not templates. A single pod resource will be realized into a single set of the described containers. It does not make sense to talk about "scaling" a pod. You are certainly free to create a bunch of copies of a pod (by literally creating multiple pod definitions), but there are better ways to accomplish this.

### Exercise 1: Explain Pods and Services

Before you can get started working with Pods you need to know about their structure and associated behaviors. You could pull up the Kubernetes API reference documentation but the documentation is also built into Kubernetes itself. The `explain` subcommand will describe any resource or resource property for any version of that resource supported by the current platform.

Run `kubectl explain pod` to see the top-level structure of a Pod. The `Status` property is special in Kubernetes and is read-only. When you're defining a Pod with YAML each of the listed properties (with the exception of Status) should be specified. You should explain the `pod.spec` and `pod.spec.containers` to see what properties are available.

Kubernetes defaults to showing the earliest version of a resource available on the platform. You must specify the `--api-version` flag to lookup any specific version of a resource.

Use `kubectl explain --api-version v1 pod`, `kubectl explain --api-version v1 pod.spec` and `kubectl explain --api-version v1 pod.spec.containers` to consider a **v1 Pod** that runs Redis in a container. The Pod might be configured as follows:

* Pod name: `redis`
* Pod label key: `app` and value: `redis`
* Single container named `redis`, using the `redis:alpine` image.

There isn't going to be quite enough time to write your own in this workshop, but consider the solution at `./workload/redis-pod.yaml`. 

```
# Everything in this file is part of the same YAML document and 
# thusly the same Kubernetes resource definition.

apiVersion: v1    # Kubernetes API version
kind: Pod         # Document resource kind

metadata:         # Resource-level metadata
  name: redis     # Name of the resource
  labels:         # Applying taxonomy allows other resources to 
                  # refer to this via selectors.
    app: redis    # Labels are in a <key>: <value> form.
    version: v1   # The version label is not special, though 
                  # it is common for use with deployment and 
                  # routing selection.
  # This will keep solution resources in a separate Kubernetes namespace
  namespace: workshop

# Every resource has a spec. This is a free-form property map 
# and validated on the serverside to match known resource types.
spec:            

  # Each pod can contain multiple containers. They all share IPC, 
  # NET, and UTS namespaces. They are all on the same "virtual host."
  containers:    
   - name: redis
     image: redis:alpine
```

Kubernetes provides Namespaces to isolate collections of related resources. A Namespace is another resource type and is created, and otherwise managed the same way as any other Kubernetes resource. By default resources are created in the `default` Namespace. That is where most of your work today will be created. 

All resource names must be unique within any give Namespace. That is to say you cannot have two Pods named, `redis` in the same Namespace. So we want to keep the provided solutions separate from your workspace. To do so we've defined a new Namespace called, `workshop`. You can see that definition in [custom-ns.yaml](./workload/custom-ns.yaml).

Next, you'll launch the provided solutions, but make sure you `create` the namespace first: 

```
kubectl create -f workload/custom-ns.yaml
```

### Exercise 2: Launch and a Redis Pod

Next you're going to start your Redis pod. The `kubectl apply` subcommand uses declarative tooling to analyze the delta between the state on the platform and the new desired state. You'll use this subcommand and specify the file containing your Pod definition. Do that now.

```
kubectl create -f workload/redis-pod.yaml
```

Once your Pod has been created you should be able to see it and any other Pods running on your platform. Do that now by running `kubectl get pod --namespace workshop`. 

The output should look something like:

```
NAME      READY     STATUS    RESTARTS   AGE
redis     1/1       Running   0          2m
```

This is a list of the Pods running in the workshop Namespace. Note that 1/1 is in a Ready state. For reference you can compare that list with the Pods running in the `default` Namespace by running `kubectl get po` (po is short for pod). There shouldn't be anything listed.

So, Redis is running "somewhere" but how do you access it? How do you know anything about it? You can get deep details about any Kubernetes resource (including your Pods) with the `kubectl describe` subcommand. Like other kubectl commands the first argument is always the resource type that you're inspecting and the remainder is a variadic list of names that you want to operate on. 

```
kubectl describe po redis --namespace workshop
```

In the output from describe you'll see associated metadata, relationships with other resources, the set of associated containers, and an assigned IP address (this is the ClusterIP which we'll cover later). This is the IP address for the Pod. However you'll also note that it is a private network IP address that is not likely routable from your browser or terminal. If you want to test this Redis instance you're going to need to get creative.

You might be familiar with Docker already and be used to the idea of execing a command in a running container. Well Kubernetes is built on top of Docker or other similar container technology. You can see the redis container in your Pod by running `docker ps`.

You could exec a command in the redis container, but in production you'd only be able to do that from the cluster node where the container is running. That won't scale well. Instead use the `kubectl exec` subcommand. This will execute a command inside the container. It is similar in form and function but since Pods can contain multiple containers it does require a bit of refinement. Use the following command to define and increment a counter in your redis instance:

```
kubectl exec redis --namespace workshop -i -t -- redis-cli incr my-training-counter
``` 

Repeat that command a few times to watch the counter increase.

By default the exec subcommand will use the first container defined in the Pod spec. If you need to exec on a different container you should specify both the Pod and container name:

```
kubectl exec redis -c redis --namespace workshop -i -t -- redis-cli incr my-training-counter
```

When you're finished having fun with your redis instance tear down your Pod with the `kubectl delete` subcommand. Remember, you specify the resources to delete by pointing to the resource definition files (the `-f` flag). 

### Exercise 3: Compose to Kubernetes Conversion with Pods and Services

Now lets start working with a real application. It is time to create resources to launch the voting application in Kubernetes. 

This software in this exercise involves multiple processes that use each other over the network. Since this example was originally designed for use in Docker you're going to need to use a few tricks to make it work. 

You're going to create two pods. For now it might be easier to put them both in the same file. To do so separate the resource definitions with the YAML document separator `---` on its own line. The two pods should be defined as follows:

##### Pod 1: Voter

Should be named `voter` and have labels `app: voter` and `version: v1`. The pod spec should include the following container definitions:

| Container name  | Port | Image |
| --------------- |:----:| -----:|
| redis | 6379 | `redis:alpine` |
| vote | 80 | `dockersamples/examplevotingapp_vote:before` |

When software in this pod goes to resolve the name, "redis" it should get "127.0.0.1" as the result.

##### Pod 2: Results

Should be named `results` and have labels `app: results` and `version: v1`. This time use a volume to isolate the database state. The pod spec should include the following volume definitions:

| Name  | EmptyDir |
| --------------- |:----:|
| db-data | `{}` | 

And the following container definitions:

| Container name  | Port | VolumeMounts | Image |
| --------------- |:----:|:------------:| -----:|
| db | 5432 | `name: db-data` `mountpoint: /var/lib/postgresql/data` | `postgres:9.4` |
| worker | NA |  | `dockersamples/examplevotingapp_worker` |
| result | 80 |  | `dockersamples/examplevotingapp_result:before` |

When software in this pod goes to resolve the name, "db" it should get "127.0.0.1" as the result.

Consider the solution at [votingapp-pods.yaml](./workload/votingapp-pods.yaml).

The five services are split in this way because the vote and results container both have software that binds to port 80. Those applications cannot live in the same pod. The act of voting and result publishing is bound by an asynchronous data processing pipeline and in this case the redis instance is a synchronous dependency for the voting application, but not the results application. By splitting the architecture into these two pods we can be sure that each component will function correctly without the other.

After you've created the pod definitions you need to provide the service discovery glue so that they can talk to each other and so you can start voting.

##### Services

You should know that every pod gets an IP address, but pods are not automatically discoverable via DNS. Network service discovery and load balancing functionality are provided by Service resources. 

A Service resource is little more than a description of a reverse proxy. Kubernetes will publish specific DNS records for every Service resource, and realize proxies in according to the Service spec. Take a moment to checkout the documentation for the `service` resource and its `spec` property.

```
kubectl explain service
kubectl explain service.spec
```

Take note of three important `service.spec` fields: type, selector, and ports.

There are several Service types. Some types are for internal service definitions, others are for integration with broader systems like a cloud environment or managed DNS system. The default type is `ClusterIP` which creates an internal IP address that is serviced by a reverse-proxy component called, `kube-proxy.` ClusterIP provides lightweight internal network load balancing (layer 4).

Since you're running these examples on your local machine and a single-node Kubernetes cluster it will be easiest to use the `NodePort` service type for services that you want to access outside of Kubernetes. That type behaves similar to Docker port publishing and works well for local testing. 

Create Service resource definitions according to the following table:

| Service name  | Publish Port | Internal (y/n) | Selector |
| ------------- |:------------:|:--------------:|---------:|
| redis | 6379/TCP | y | `app: voter` |
| vote | 31000/TCP | n | `app: voter` |
| result | 31001/TCP | n | `app: results` |

Consider the solution at [services.yaml](./workload/services.yaml).

Once you're done startup your app: `kubectl create -f ./workload/votingapp-pods.yaml -f ./workload/services.yaml`

##### Trying it out

Poll the resources a few times with `kubectl get po` and `kubectl get svc`. When all of the resources are ready you should be able to load up the voting and results apps in your web browser by navigating to [localhost:31000](http://localhost:31000) and [localhost:31001](http://localhost:31001) respectively. 

Votes cast in one window should be reflected in the other shortly.

Don't forget to tear down the solutions when you're done with `kubectl delete -f ./workload/votingapp-pods.yaml -f ./workload/services.yaml`

## Resilience and Automated Deployment Control 

Pods are fine, but working with them is a bit like working with raw containers in Docker. They provide a great language for describing workloads, but they do little to help us abstract workflows. Upgrading a service using raw Pod resources would involve manually creating new Pod versions, updating some label like `version` and then updating Service selectors to route to the new and old pods, and then manually take down old versions. Yuck. Processes like that are formulaic and should be automated whenever possible. Leaving them un-automated only introduces opportunities for human error. Kubernetes provides those automations (and many more powerful ones) but adopting them means working with higher-level resource types. 

The next resource you'll work with is called a ReplicaSet. 

### Exercise 4: Resiliency with ReplicaSets

ReplicaSets provide replica and resilience controls. They create Pod resources on your behalf according to the template and other properties you provide in the ReplicaSet spec. If a Pod under management of a ReplicaSet is deleted or fails then the ReplicaSet will take care of replacing it and maintaining the realized desired state of the system.

**Note**: there is another resource type called a ReplicationController that is deprecated. New users and workloads should use ReplicaSets or other higher-level resources.

Now create a new file to describe your stack in terms of ReplicaSets and Services instead of raw Pod resources. Call the new file `voting-rs.yaml`. Include all of the ReplicaSet and Service definitions in that same file.

The ReplicaSet resource type is defined in the `apps/v1` API version. The `apps` API tree is full of higher-level abstractions.

You're going to want to create two ReplicaSets, one for each type of Pod in the previous exercise. Use `kubectl explain replicaset --api-version apps/v1` and `kubectl explain replicaset.spec --api-version apps/v1` to get the structure.

The replicaset solutions are in [votingapp-replicasets.yaml](./workload/votingapp-replicasets.yaml). Start the solution now by running:

```
kubectl apply -f ./workload/votingapp-replicasets.yaml -f ./workload/v2-services.yaml
```

The `apply` command is a full declarative tool provided by the CLI. It does the hard work of examining the resources described in the YAML, determining if they exist in the database, and then merging the changes with the current state in the database if they do exist. Using `apply` during resource creation will enable you to use `apply` to make changes to the resources later.

When they've reached a full ready state open the voting application in your web browser: [localhost:31000](http://localhost:31000). Cast a vote and note the "container ID" at the bottom of the page. This is the hostname of the pod where the request was served. Note that you did not specify that name anywhere. It was created by the ReplicaSet. Go lookup the running Pod and ReplicaSet resources.

```
kubectl get rs --namespace workshop
kubectl get po --namespace workshop
```

Use `kubectl describe rs` to see the list of events associated with the voter ReplicaSet. You should see a line like near the bottom:

```
Normal  SuccessfulCreate  1m    replicaset-controller  Created pod: voter-f9599
```

Events are just resources in Kubernetes. This is a list of all the events created by the replicaset-controller component in relation to this ReplicaSet resource. Now put the ReplicaSet resilience to the test. Delete the pod that served the last request. Using the Pod name from the event above you'd issue a command like, `kubectl delete po voter-f9599`.

Deleting the Pod is one type of failure from the perspective of the replicaset-controller. Since the ReplicaSet resource remains in the system and now the realized state is our of spec from the desired state it should create a replacement Pod immediately.

You should describe the ReplicaSet resource again and examine the Event list. You'll see that there is a new event describing the creation of a new Pod. If you then reload the voting page you'll notice that the app still works and is being served by the replacement Pod.

You've witnessed both the resilience and dynamic service backend routing mechanisms provided by Kubernetes controllers. Before moving on, take a moment to vote Cats vs Dogs. Then load the results page [localhost:31001](http://localhost:31001) and verify that the vote was recorded.

The problem with containers, Pods, ReplicaSets, etc is that they are primarily concerned with runtime context and consistent disposable processes. Persistent state needs to be modeled. The results Pod uses a volume, but a Kubernetes volume is scoped to a Pod. Volumes are cleaned up when Pods are cleaned up and they are not shared across Pods. Watch what happens when you delete your results Pod. Do that now and reload the results page. You should note the vote goes back to 50/50.

Pods are replaced all the time as part of natural life cycle events like deployments or cluster node replacement. Before you learn how to deal with persistent state you should be familiar with some life cycle workflows.

### Other exercise 5: A Life Cycle Adventure

Performing a rolling deployment by hand on any platform usually goes something like:

* Verify that some minimum percentage of the service replicas are in a healthy state
* Identify one or some batch of replicas to replace
* Remove those replicas from the reverse proxy backend pool
* Stop those replicas
* Start a new replica of with the updated configuration for each stopped replica
* Verify that each new replica enters a running or ready state
* Add the new replias to the reverse proxy backend pool
* Monitor service health for some period, repeat from step 1 until all replicas have been replaced
* Mark the deployment as complete and clean up old replicas

ReplicaSets maintain a desired number of Pods, but don't provide any deployment workflow. To get an appreciation for that workflow you're going to deploy an update to your ReplicaSet definitions from the last example.

Product management issued you a ticket and they want to roll out a new poll. They'd like to stop asking people about Cats vs. Dogs and focus more on the enterprise software crowd. They want to poll Java vs. .NET. Luckily, that software has already been written and is available simply by updating the image tag.

**With your solution from exercise 4 running**, update your definition file and change the images used for the voter and results containers from the `:before` tag to the `:after` tag. When you've updated both Pod definitions you're going to trigger a deployment using the `kubectl apply` command.

Once the changes have been applied reload [localhost:31000](http://localhost:31000) and [localhost:31001](http://localhost:31001). Did you see where the new poll came up? No? What if you refresh again? Maybe wait a few seconds and try again. No? You might notice that the vote page is being served by the same Pod as before. It is almost like no changes were made to the running software.

Well if you inspect the ReplicaSet configurations with `kubectl describe` you'll note that the Pod template does specify the `:after` image. But that there is only a single Pod creation event in the history. It looks like you'll need to manually trigger the replacement. 

Delete the Pod that was created by the voter ReplicaSet, check the description again and verify that a new Pod was created and reload the page. Viola. In this case the Service definition you used for the voter and redis services only select backends based on the `app` label. As updated version come into service they are automatically included in the Service definition so no manual proxy adjustment was required.

Delete the result Pod to flip that side of the app as well.

This was a dangerous flip operation. You could have done a more seamless migration here by creating new ReplicaSet definitions with an increased `version` label and carefully adjusting the `replias` property on each. But either way manipulating these resources manually is error prone.

**Before moving on** cleanup all of the pods, replica sets, and services that you created for this exercise.

### What is Next?

The workflow described in the previous exercise is automated by Kubernetes Deployment resources. Using those, diving into batch processing with Jobs, and covering state management with PersistentSets are covered by the full workshop offered by [Topple](https//www.gotopple.com). If you interested in learning more about workload resources and management workflows using Kubernetes reach out and ask about our multi-day workshops.


