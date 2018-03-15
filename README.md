This content has also been [published to my blog](https://blog.stigok.com/post/kubernetes-non-dynamic-persistent-volumes-and-claims).

---


I was attempting to deploy a Docker Compose file with `kompose up -f docker-compose.yml`. This compose file has a named volume definition which, when using `docker-compose up` will automatically create a persistent volume on the host machine, but not so easily with `kompose up` in my Kubernetes cluster with the default configuration.

This is an example service which doesn't even make use of the mounted volume, but I don't think it really matters. Let's just start going through the process of deploying it to Kubernetes by looking at the initial Docker Compose file:

```
$ cat docker-compose.yml
version: "2"

services:
  pinger:
    image: alpine
    command: ["ping", "stigok.com"]
    volumes:
      - pinger-files:/data

volumes:
  pinger-files:
```

And this is all good when deploying locally with `docker-compose`:

	$ docker-compose up
    docker-compose up
    Starting gcek8spvctest_pinger_1 ... done
    Attaching to gcek8spvctest_pinger_1
    pinger_1  | PING stigok.com (51.15.90.218): 56 data bytes
    pinger_1  | 64 bytes from 51.15.90.218: seq=0 ttl=49 time=26.582 ms
    pinger_1  | 64 bytes from 51.15.90.218: seq=1 ttl=49 time=26.303 ms
    ^CGracefully stopping... (press Ctrl+C again to force)
    Stopping gcek8spvctest_pinger_1 ... 
    Killing gcek8spvctest_pinger_1 ... done

 but when using `kompose`, it is ignoring the root `volumes` definition and simply creates a PersistentVolumeClaim (PVC) for the volume `pingerfiles`. To demonstrate, let's first creating a separate namespace:

	$ kubectl create namespace pinger
	namespace "pinger" create

Then deploy directly to the cluster

	$ kompose up --namespace pinger 
	WARN Unsupported root level volumes key - ignoring 
	INFO We are going to create Kubernetes Deployments, Services and PersistentVolumeClaims for your Dockerized application. If you need different kind of resources, use the 'kompose convert' and 'kubectl create -f' commands instead. 
	 
	INFO Deploying application in "pinger" namespace  
	INFO Successfully created Service: pinger         
	INFO Successfully created Deployment: pinger      
	INFO Successfully created PersistentVolumeClaim: pinger-files of size 100Mi. If your cluster has dynamic storage provisioning, you don't have to do anything. Otherwise you have to create PersistentVolume to make PVC work 

	Your application has been deployed to Kubernetes. You can run 'kubectl get deployment,svc,pods,pvc' for details.

As one of the `INFO` messages returned above states;
> Successfully created PersistentVolumeClaim: pinger-files of size 100Mi. If your cluster has dynamic storage provisioning, you don't have to do anything. Otherwise you have to create PersistentVolume to make PVC work

Checking the status of the deployment:

	$ kubectl --namespace pinger get deployment,svc,pods,pvc
	NAME            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
	deploy/pinger   1         1         1            0           2m

	NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)     AGE
	svc/pinger   ClusterIP   None         <none>        55555/TCP   2m

	NAME                         READY     STATUS             RESTARTS   AGE
	po/pinger-867cd7f488-cm4c4   0/1       CrashLoopBackOff   3          2m

	NAME               STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
	pvc/pinger-files   Bound     pvc-ccd04385-2858-11e8-9c94-42010a840032   1Gi        RWO            standard       2m

So it seems that the PVC has in fact been sucessfully created, and it has, but it is not mapping correctly. If I look in the Cluster Management UI reachable through `kubectl proxy`, it shows a more detailed error message:

![Cluster Event Log for namespace "pinger"](https://public.stigok.com/img/1521122796042501549.png)

Since I am using the Default Storage Class in our Kubernetes managed cluster in Google Cloud Engine (GCE), my default is static provisioning. So in order for a PVC to be successfully created and come up, I must first create a persistent Disk in GCE.

	$ gcloud compute disks create --size 10GB pinger-test-disk
	WARNING: You have selected a disk size of under [200GB]. This may result in poor I/O performance. For more information, see: https://developers.google.com/compute/docs/disks#performance.
	Created [https://www.googleapis.com/compute/v1/projects/broentech-test-cluster/zones/europe-west1-c/disks/pinger-test-disk].
	NAME              ZONE            SIZE_GB  TYPE         STATUS
	pinger-test-disk  europe-west1-c  10       pd-standard  READY

	New disks are unformatted. You must format and mount a disk before it
	can be used. You can find instructions on how to do this at:

	https://cloud.google.com/compute/docs/disks/add-persistent-disk#formattin

Formatting the volume is out of scope, so follow the link for detailed information. Short story is I created a VM, mounted the disk, partinioned it with `fdisk`, then created an *ext4* file system with `mkfs.ext4`.

Now create a Persistent Volume (PV) which will be mapped to the new *pinger-test-disk*. Below is the file describing the PV to be created. Note the `gcePersistenDisk` definition which mirrors the settings of the disk.

	$ cat pinger-files-pv.yaml
	kind: PersistentVolume
	apiVersion: v1
	metadata:
	  name: pinger-files-pv
	  labels:
	    type: local
	spec:
	  capacity:
	    storage: 10Gi
	  storageClassName: standard
	  accessModes:
	    - ReadWriteOnce
	  gcePersistentDisk:
	    pdName: pinger-test-disk
	    fsType: ext4



Note `storageClassName: standard` is required if a default storage class has not been set on the Kubernetes Cluster. Otherwise the PVC will be stuck in `Pending` state indefinitely, hence the deployment will never succeessfully go up.

Create the PV:

	$ kubectl --namespace pinger create -f pinger-files-pv.yaml 
	persistentvolume "pinger-files-pv" created

Query the PV to see its status:

	$ kubectl get pv pinger-files
	NAME           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
	pinger-files-pv   10Gi       RWO            Retain           Available             standard                 8m

Now, take down the old deployment before continuing.

	$ kompose --namespace pinger down
	WARN Unsupported root level volumes key - ignoring 
	INFO Deleting application in "pinger" namespace   
	INFO Successfully deleted Service: pinger         
	INFO Successfully deleted Deployment: pinger      
	INFO Successfully deleted PersistentVolumeClaim: pinger-files

Convert the Compose file into yaml files by using `kompose`

	$ kompose convert
	WARN Unsupported root level volumes key - ignoring 
	INFO Kubernetes file "pinger-service.yaml" created 
	INFO Kubernetes file "pinger-deployment.yaml" created 
	INFO Kubernetes file "pinger-files-persistentvolumeclaim.yaml" create

Before continuing, in *pinger-files-persistentvolumeclaim.yaml*, a new property has to be added as a child of `spec` to specify the `volumeName` of our PV (*pinger-files-pv*). After the update, it should look something like the following:

	$ cat pinger-files-persistentvolumeclaim.yaml 
	apiVersion: v1
	kind: PersistentVolumeClaim
	metadata:
	  creationTimestamp: null
	  labels:
	    io.kompose.service: pinger-files
	  name: pinger-files
	spec:
	  accessModes:
	  - ReadWriteOnce
	  resources:
	    requests:
	      storage: 100Mi
	  volumeName: pinger-files-pv
	status: {}

Now create the resources one after another using `kubectl` instead of `kompose`

	$ kubectl --namespace pinger create -f pinger-files-persistentvolumeclaim.yaml 
	persistentvolumeclaim "pinger-files" created

	$ kubectl --namespace pinger create -f pinger-service.yaml 
	service "pinger" created

	$ kubectl --namespace pinger create -f pinger-deployment.yaml 
	deployment "pinger" created

List the status of the resources

	$ kubectl --namespace pinger get all
	NAME            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
	deploy/pinger   1         1         1            1           21m

	NAME                   DESIRED   CURRENT   READY     AGE
	rs/pinger-669d959f5c   1         1         1         21m

	NAME            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
	deploy/pinger   1         1         1            1           21m

	NAME                   DESIRED   CURRENT   READY     AGE
	rs/pinger-669d959f5c   1         1         1         21m

	NAME                         READY     STATUS    RESTARTS   AGE
	po/pinger-669d959f5c-ng5l2   1/1       Running   0          21m

	NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)     AGE
	svc/pinger   ClusterIP   None         <none>        55555/TCP   23m

Now it looks good again.

## Aftermath

Now -- this was a lot of work. I suspect that the way to go is to enable dynamic allocation of PV's so you don't have to go through these hurdles. I want this to be as easy as simply using `docker-compose up` command with `kompose`, but I don't have a lot of experience with Kubernetes and the CLI's yet, so it may be that I'm missing something obvious here.

Please leave a comment if you have any tips!

## References
- https://cloud.google.com/kubernetes-engine/docs/tutorials/persistent-disk
- https://stackoverflow.com/questions/44891319/kubernetes-persistent-volume-claim-indefinitely-in-pending-state
- https://docs.openshift.org/latest/dev_guide/persistent_volumes.html
- https://kubernetes.io/docs/tools/kompose/user-guide/
