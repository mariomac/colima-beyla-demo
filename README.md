# Beyla on Mac with Virtual Machines and Kind: quick walkthrough


Requirements:

* Rancher Desktop (https://rancherdesktop.io/) or 
    - Colima used to work but latest version downloads images with BPF FS disabled.
    - Docker Desktop might fail for some concrete featuresk 
* Kubectl https://kubernetes.io/es/docs/reference/kubectl/
* Kind https://kind.sigs.k8s.io/

Easy install on Mac:

```
brew install podman
brew install kubectl
brew install kind
```

## Step 1. Install Docker inside a virtual machine

If you are using Rancher Resktop, just run it.

```
limactl start template://docker  --cpus 4 --disk 50 --memory 4
docker context create lima-docker --docker "host=unix:///Users/mmacias/.lima/docker/sock/docker.sock"
docker context use lima-docker
```

Once the process finishes, tests that it correctly works:

```
$ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
478afc919002: Pull complete
Digest: sha256:1408fec50309afee38f3535383f5b09419e6dc0925bc69891e79d84cc4cdcec6
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
....
```

## Step 2. Create a single-node kind cluster

```
$ kind create cluster
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.30.0) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a nice day! 👋
```

Once the process finished, check that you can have access to the cluster:

```
$ kubectl get pods -A
NAMESPACE            NAME                                         READY   STATUS    RESTARTS   AGE
kube-system          coredns-7db6d8ff4d-2km9s                     1/1     Running   0          47s
kube-system          coredns-7db6d8ff4d-xw9nr                     1/1     Running   0          48s
kube-system          etcd-kind-control-plane                      1/1     Running   0          63s
kube-system          kindnet-kqkcw                                1/1     Running   0          48s
kube-system          kube-apiserver-kind-control-plane            1/1     Running   0          63s
kube-system          kube-controller-manager-kind-control-plane   1/1     Running   0          63s
kube-system          kube-proxy-4952t                             1/1     Running   0          48s
kube-system          kube-scheduler-kind-control-plane            1/1     Running   0          63s
local-path-storage   local-path-provisioner-988d74bc-rqgx4        1/1     Running   0          47s
```

## Step 3. Deploy a test application and Beyla to instrument it

On Kubernetes, we recommend using Helm to deploy Beyla:

But for simplicity purposes we will manually deploy the files included in this repository. Don't forget to download this repo fro Github first.

```
kubectl apply -f basic/serviceaccount.yml
kubectl apply -f basic/config.yml
kubectl apply -f basic/sampleapps.yml
kubectl apply -f basic/daemon3.yml
```

Check that the application pods and Beyla pods are deployed:

```
$ kubectl get pods -A
NAMESPACE            NAME                                         READY   STATUS    RESTARTS   AGE
beyla                beyla-sp4xv                                  1/1     Running   0          52s
default              docs-67c7dcbb4d-65h4b                        1/1     Running   0          52s
default              docs-67c7dcbb4d-kfmtr                        1/1     Running   0          52s
default              website-6fb65bcb8d-gj6bc                     1/1     Running   0          52s
default              website-6fb65bcb8d-kfkbh                     1/1     Running   0          52s
...
```

Check that Beyla started correctly and keep tailing the logs for later checks:

(replace `beyla-sp4xv` by your own Beyla instance name)

```
$ kubectl logs -n beyla -f beyla-sp4xv
Defaulted container "beyla" out of: beyla, mount-bpf-fs (init)
time=2024-07-30T09:55:10.648Z level=INFO msg="Grafana Beyla" Version=cc5a468f Revision=cc5a468f "OpenTelemetry SDK Version"=1.24.0
time=2024-07-30T09:55:10.649Z level=INFO msg="starting Beyla in Application Observability mode"
time=2024-07-30T09:55:10.752Z level=INFO msg="Detected mounted BPF filesystem at /custom-bpf/beyla-beyla" component=discover.TraceAttacher
time=2024-07-30T09:55:10.753Z level=INFO msg="using hostname" component=traces.ReadDecorator function=instance_ID_hostNamePIDDecorator hostname=beyla-spznv
time=2024-07-30T09:55:10.754Z level=INFO msg="Starting main node" component=beyla.Instrumenter
time=2024-07-30T09:55:10.977Z level=INFO msg="instrumenting process" component=discover.TraceAttacher cmd=/usr/local/apache2/bin/httpd pid=4062
time=2024-07-30T09:55:10.977Z level=INFO msg="new process for already instrumented executable" component=discover.TraceAttacher pid=3698 child=[3717] exec=/usr/local/apache2/bin/httpd
time=2024-07-30T09:55:10.977Z level=INFO msg="new process for already instrumented executable" component=discover.TraceAttacher pid=3698 child=[] exec=/usr/local/apache2/bin/httpd
time=2024-07-30T09:55:10.977Z level=INFO msg="new process for already instrumented executable" component=discover.TraceAttacher pid=3939 child=[] exec=/usr/local/apache2/bin/httpd
time=2024-07-30T09:55:10.978Z level=INFO msg="new process for already instrumented executable" component=discover.TraceAttacher pid=4062 child=[4082] exec=/usr/local/apache2/bin/httpd
time=2024-07-30T09:55:10.978Z level=INFO msg="new process for already instrumented executable" component=discover.TraceAttacher pid=3939 child=[3958] exec=/usr/local/apache2/bin/httpd
time=2024-07-30T09:55:10.978Z level=INFO msg="new process for already instrumented executable" component=discover.TraceAttacher pid=3818 child=[] exec=/usr/local/apache2/bin/httpd
```

```
$ kubectl port-forward service/docs 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

And do some requests to http://localhost:8080:

```
curl http://localhost:8080
```

The beyla logs should now show a trace like this for each request:

```
2024-07-30 10:19:20.730101920 (280.541µs[280.541µs]) HTTP 200 GET / [127.0.0.1 as 127.0.0.1:57678]->[127.0.0.1 as 127.0.0.1:80] size:77B svc=[default/docs generic] traceparent=[00-61b6e41efa065e988844cc178b3d5fb8-0000000000000000-01]
```

This means that it was able to instrument your application!


