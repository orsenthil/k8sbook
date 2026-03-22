Kubernetes workloads are containers that are running on the machines. The containers are instances of an application created from a container image. Container images are bundled with the application and its dependencies. We build container images with a Dockerfile and its dependencies. The Dockerfile is how we define the container image that runs as the workload.

The lifecycle of the pod from the docker image to a running container goes like this. When you run your application, you communicate with the API Server, which is called the Control Plane, and you ask the API Server to run your application, which is called a Pod on the Worker Node.

When updating the application, very often run into a issue where the pod is struck in the Pending State and hasn't transitioned to running state yet.

This is the lifecycle of pod when the when the deployment is updated to the point when it is run.


![](disk-pressure-1.png)


The control flows through the API Server to the Scheduler and to the Kubelet. When you update your deployment, the API server accepts the new podspec. PodSpec is a type of resource defined in the Kubernetes objects.  
When we update our application in the Kubernetes cluster, which is a Deployment, the API Server accepts a new PodSpec, the scheduler picks up a node that meets the criteria for running the application. Updating the deployment usually means it picks the nodes where the pods are already running, and it needs to update the image of the currently running pod.

The Scheduler picks the node and the Kubelet on the node receives the pod assignment via its watch on the API server. The kubelet's `syncPod` loop kicks in.  In the end, the Kubelet's `syncPod` method calls into the container runtime to ensure that the pods containers are running.  Before starting any container, it must ensure that the image of the container is present.

Kubelet does not pull the images itself. It calls another service called Container Runtime Interface (CRI) `PullImage` Remote Procedure Call.

The `imagePullPolicy` that we define in the YAML files determines how the Kubelet's image manager will pull the image.
The `imagePullPolicy` matters enormously here. With `Always` (default for `:latest`), it always contacts the registry. With `IfNotPresent`, it checks the local image store first.

The next stage in the pipeline is where containerd resolves and pulls the image layers. containerd, in most cases, and sometimes a program called CRI-O, receives the `PullImage` RPC from Kubelet, and it resolves the image tag to a digest, and pulls each layer.
In the image pull, each layer is downloaded as a compressed blob over HTTPS, then decompressed and unpacked to disk. The key path on the disk is `/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/`

There is a process called Snapshotter. The snapshotter creates a directory per layer. When all the layers are ready, it mounts them as a union file system, the lower layers are read-only and the container gets a thin writable layer on the top.


When you update your application and the worker node does not have enough disk capacity to accommodate the uncompressed image, the application fails to update. The previous images and containers will still be running, but the new ones would not have been updated.   



![](disk-pressure-2.png)



And imagine if your application is served by hundreds of pods on the node, and suddenly a few of the containers will try to download and will be stuck in "ErrImagePull".

Your new pods will be in the Pending state and only on close inspection will you see the "reason": "ErrImagePull".



```json
{
  "kind": "Pod",
  "apiVersion": "v1",
  "metadata": {
    "name": "gpu-service-7f8d9c6b4-xk2pn",
    "namespace": "production"
  },
  "status": {
    "phase": "Pending",
    "conditions": [
      {
        "type": "PodScheduled",
        "status": "True"
      }
    ],
    "containerStatuses": [
      {
        "name": "gpu-service",
        "state": {
          "waiting": {
            "reason": "ErrImagePull",
            "message": "rpc error: code = Unknown desc = failed to pull and unpack image: write /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/4192/fs/usr/lib/x86_64-linux-gnu/libcuda.so.535.129.03: no space left on device"
          }
        },
        "image": "your-registry.ecr.us-east-1.amazonaws.com/gpu-service:v2.3.1-ubuntu",
        "imageID": ""
      }
    ]
  }
}
```


And the kubelet events can give you the cause.

And the **events** you would see:

```json
{
  "kind": "Event",
  "apiVersion": "v1",
  "metadata": {
    "name": "gpu-service-7f8d9c6b4-xk2pn.17e2a3b4c5d6e7f8"
  },
  "involvedObject": {
    "kind": "Pod",
    "name": "gpu-service-7f8d9c6b4-xk2pn"
  },
  "reason": "Failed",
  "message": "Failed to pull image \"your-registry/gpu-service:v2.3.1-ubuntu\": rpc error: code = Unknown desc = write /var/lib/containerd/...: no space left on device",
  "type": "Warning",
  "count": 3,
  "source": {
    "component": "kubelet",
    "host": "ip-10-0-1-47.ec2.internal"
  }
}
```



When working with GPU Services, the images often are very large in size. Not ensuring that nodes will have sufficient capacity will often put your workloads in the Pending state with Disk Pressure issues.
These are some of the easiest to troubleshoot, but resolution time is not quick here. You will have to go back to source to rebuild the application and ensure that your image is constrained to an acceptable size, rebuild and push to the registry, update your Helm charts or manifests and then apply it again.

This process can be time consuming to resolve. In production critical scenarios, we have resolved to increase the disk size of the worker nodes quickly by attaching additional volume mounts so that worker nodes are given additional capacity.

One of the best approaches I have come across in solving these issues is to have a CI check which can detect if the image size has increased a lot. This can save a lot of pain later down the road.



![](disk-pressure-3.png)
