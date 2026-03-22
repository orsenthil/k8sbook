When we are troubleshooting kubernetes, we often see this in the logs.

```
E0317 12:34:56.789 leaderelection.go:367] failed to renew lease: context deadline exceeded
```

What has happened is the Controller has failed to renew the lease and is not able to serve your request.   Well that is pretty much obvious from the error message itself.  But let us see it in a bit more detail.

Kubernetes Controllers are the manifestation of the control loop mechanism in action for the cluster state.  Just like a thermostat in the room, which powers on or off to bring the temperature of your room to the desired level.

The kubernetes controllers bring the state of the cluster to the one defined in your yaml files. You tell Kubernetes "I want 3 replicas of my app." The controller's job is to constantly check: "Are there 3 replicas running right now?" If not, it creates or kills pods until reality matches the wish.  This pattern is called reconciliation, closing the gap between the desired state and the actual state.
When you update a deployment in Kubernetes, it kills the previously running pods and brings up the number of pods to the required number we want. This is the job of the deployment controller.

![](leader-election1.png)


Kubernetes Control Plane comes with the default set of controllers that are responsible for the basic things like creating a job, running your deployment, creating the replica set.  You can see those control plane controllers if you run your own Kubernetes cluster.

We often tend to extend Kubernetes by running our own controllers too, which work on specific type of jobs. Those can be run on the Control Plane or at the application level on the data plane.
The controllers we build are built or extended using the `controller-runtime` package, which is a wrapper over `client-go` where the logic to capture the lease and become the leader is encoded.
Any controller using `controller-runtime` / `client-go` doesn't just run, it first competes for a lease object stored in the etcd via the API server. Only the pod holding the lease runs the reconcile loop. Everything else waits.


![](leader-election2.png)

Inside each of the Kubernetes Controller there are few processes.

There is an Informer, a long-lived watch connection to the API Server. It maintains a local cache of all the objects the controller cares about. The pods, nodes, all the custom resources. When something changes, it doesn't re-list everything, it receives a stream of events.
The informer does not call the reconcile function directly. Instead, it drops the object's key (like the default/my-pod) into a rate limited work queue. This gives the deduplication for free. If the same object changes 10 times a second, the reconciler only runs once because of the queue.  

The reconcile function is the main part. It receives the Request containing the object's name / namespace. It reads the current state from the cache (lister), compares it to the desired state, and makes API calls to fix any drift. If it returns an error, the key goes back in the queue with exponential back off.

Leader election is how Kubernetes ensures only one instance of a controller is actively doing work at any time, even if multiple replicas are running.
The leader election is the mechanism for the locking in our concurrency scenario. Only one leader writes to ensure consistency of writes.  Without single-leader, every reconcile action risks being duplicated.  






![](leader-election3.png)

In the entire leader election process, the pods try to get the lease object and inform the API server that the particular pod is the leader and it will be doing the operations. It constantly renews the lease for its leadership, and after lease expiry another pod of the same controller process can become the leader too.

In a Kubernetes cluster we often run many controllers. They can be seen using

```
➜ kubectl get pods --all-namespaces -l app.kubernetes.io/component=controller

NAMESPACE       NAME                                        READY   STATUS    RESTARTS      AGE
cert-manager    cert-manager-7d678bfb4f-6hnhp               1/1     Running   1 (15d ago)   101d
ingress-nginx   ingress-nginx-controller-5dc6969574-lgf7h   1/1     Running   0             101d
```

For instance, I am running cert-manager and the ingress-nginx controller in my cluster. cert-manager provides HTTPS certificates and the ingress controller is responsible for allowing external traffic inside my cluster. It allows the public internet to reach the apps that run inside my cluster, like visiting a website or using an app on my cluster. That's the entry point there. These controllers ensure that adequate resources are present to handle the request.


And you can see the control plane controllers using

```
kubectl get pods -n kube-system -l component=kube-controller-manager
```

VPC Resource Controller was one of the controllers that runs on a Kubernetes Control Plane. It is the default in Amazon's Elastic Kubernetes Service offering, but it is open source, and customers can run it in their cluster when they want to utilize Amazon's Security Group feature at the pod level. Running into `failed to renew lease: context deadline exceeded` was something I had seen quite a few times when working with this controller. There can be multiple reasons for the leader election failure, as it is usually called.
There were many reasons for it, like the API server not being reachable due to network partitioning, a slow API server or load on etcd, lease objects deleted by mistake, or the namespace getting deleted.
Or the misconfiguration of the Role Based Access Control (permissions thing), which caused the leader election loss too. We had to update the controller with the correct permissions, and it gets running again.


![](leader-election4.png)
