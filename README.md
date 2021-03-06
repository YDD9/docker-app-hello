To use latest technology:
  * Debian9.3 stretch Linux
  * Docker17.03.0-ce
  * Kubernetes(solution Kubeadmv1.9)

Work based on host PC Win10 and VirtualBox, with different VirtualBox network settings: NAT and Bridge

* First part:
    prepare a simply Python Flask web app for Docker.
* Second part:
    deploy the app in Kubernetes Cluster(one master, one work node).

**A Docker Hello World! APP written in Python Flask deployed by kubernetes**

- [Environment](#environment)
- [List out used Python packages in requirements.txt](#list-out-used-python-packages-in-requirementstxt)
- [Dockerfile](#dockerfile)
- [Build Docker image](#build-docker-image)
- [Run app in Docker](#run-app-in-docker)
- [Verify the app in a browser](#verify-the-app-in-a-browser)
- [(Optional) Verify further the app](#optional-verify-further-the-app)
- [Push Docker image to dockerhub for easy download and reuse](#push-docker-image-to-dockerhub-for-easy-download-and-reuse)
- [(Optional) Save Docker image locally and import for use](#optional-save-docker-image-locally-and-import-for-use)
- [Use kubeadm to setup Kubernetes cluster](#use-kubeadm-to-setup-kubernetes-cluster)
- [Prepare deployment.yml and deploy app with kubectl](#prepare-deploymentyml-and-deploy-app-with-kubectl)
- [Check deployment status and troubleshot](#check-deployment-status-and-troubleshot)
    - [Once deployment created, check the process status](#once-deployment-created-check-the-process-status)
    - [Why deployment is unavailable, check deployed pods](#why-deployment-is-unavailable-check-deployed-pods)
    - [Why FailedCreatePodSandBox](#why-failedcreatepodsandbox)
    - [Why kube-flannel CrashLoopBackOff and kube-dns keep ContainerCreating...](#why-kube-flannel-crashloopbackoff-and-kube-dns-keep-containercreating)
- [kube-dns still failing and needs to be deleted](#kube-dns-still-failing-and-needs-to-be-deleted)
    - [retry with Flannel with hard coded IP range 10.244.0.0/16 and kube-dns works](#retry-with-flannel-with-hard-coded-ip-range-102440016-and-kube-dns-works)
- [Kubectl Autocomplete](#kubectl-autocomplete)
- [Deploy again the app](#deploy-again-the-app)
    - [To workaround the Linux bug persistent MAC address on VirtualBox https://github.com/systemd/systemd/issues/3374](#to-workaround-the-linux-bug-persistent-mac-address-on-virtualbox-httpsgithubcomsystemdsystemdissues3374)
    - [Error adding network: failed to set bridge cni](#error-adding-network-failed-to-set-bridge-cni)
- [Further info check other *.md files](#further-info-check-other-md-files)


# Environment
host: Win10
      VirtualBox5.2 (network setting: NAT at beginning, later switch to Bridge):
          guest: Debian9.3 stretch Linux
                 Python2.7(extra package Flask and pipreqs)
                 Docker17.03.0-ce: run inside Debian Linux
                 Kubeadm1.9
                 Kubectl
                 Kubelet


**Almost all steps are done inside the guest Debian unless specified.**

# List out used Python packages in requirements.txt
Generate a packages list based on import, ignore other not used ones.
But better to use `pip freeze` when you have a Python virtual env for dedicated projects.
```
$ pip install pipreqs
$ pipreqs --force 'C:\Users\ydd9\Documents\PythonHelloDockerK8s'
```


# Dockerfile
open dockerhub website to find a Linux image with python and follow the Dockerfile template there.
Dockerfile is used to build an image for you


# Build Docker image
```
$ cd PythonHelloDockerK8s
$ docker build ./ -t ydd9/python-hello
```


# Run app in Docker
if you execute app.py inside Debian, you will fail, because flask not installed and the goal is to run in Docker.
```
$ python app.py
missing module flask
```
when you lauch your Docker, flask will be installed inside and app.py will be executed.
```
$ cd PythonHelloDockerK8s
$ docker run -p 8888:8080 ydd9/python-hello
* Running on http://0.0.0.0:8080/ (Press CTRL+C to quit)
```


# Verify the app in a browser
open another terminal use `docker ps` to find ContainerID df573987753e of the image ydd9/python-hello,
you also notice docker container is using port 8888 and listen incoming traffic at port 8080,
just as command `docker run -p 8888:8080 ydd9/python-hello` specifies.

then use `docker inspect <ContainerID>` to find the exposed "IPAddress": "172.17.0.2" of the container

think through the scenario this way:
docker container itself will use localhost 127.0.0.1:8888 to execute the job, but inside the guest Debian, Docker app expose port 8080, so it is seen as 172.17.0.2:8080, just type `172.17.0.2:8080` in web browser inside Debian, "Hello World!" should display.

further more, if you want to see this display in host win10 web browser, what should be configured ?
VirtualBox Network settings by default use NAT, now config a port forwarding, set guest port is 8888 and any free host port we like, let's say 1234, so `127.0.0.1:1234` load the page correctly. VirtualBox route the output of docker container from port 8888 to port 1234

```
$ docker ps
CONTAINER ID        IMAGE
             COMMAND                  CREATED             STATUS              PORTS                     NAMES
df573987753e        ydd9/python-hello
             "python ./app.py"        47 seconds ago      Up 46 seconds       0.0.0.0:8888->8080/tcp   happy_lewin
4bcf36896861        gcr.io/google_containers/k8s-dns-sidecar-amd64@sha256:f80f5f9328107dc516d67f7b70054354b9367d31d4946a3bffd3383d83d7efe8         "/sidecar --v=2 --..."   5 hours ago         Up 5 hours                                    k8s_sidecar_kube-dns-6f4fd4bdf-txj26_kube-system_edd99c09-ebf4-11e7-a191-080027f0e96d_0
...

$ docker inspect df573987753e
...
"Ports": {
                "8080/tcp": null
            },
            "SandboxKey": "/var/run/docker/netns/39b487507165",
            "SecondaryIPAddresses": null,
            "SecondaryIPv6Addresses": null,
            "EndpointID": "a15143d9f57406a07b695d666158d7c64511d800113f79e77c4240c68c9d22f2",
            "Gateway": "172.17.0.1",
            "GlobalIPv6Address": "",
            "GlobalIPv6PrefixLen": 0,
            "IPAddress": "172.17.0.2",
            "IPPrefixLen": 16,
            "IPv6Gateway": "",
...

```


# (Optional) Verify further the app
With below Docker run, the app will run at port X auto chosen by Docker and you can check in Debian the display "Hello World!" at port X.
But in win10, port forwarding needs to reconfig to port X too.
```
$ docker run -p 8080 ydd9/python-hello

# find out port X
$ docker ps
```

if docker run -p publish a wrong port such as 8081, inside debian still work for `172.17.0.2:8080` but not 8081.
I think because docker run publish automatically the same port as app.py, as `docker run ydd9/python-hello` works too. inside win10 will not work, the miss match of tcp port.
```
$ docker run -p 8888:8081 ydd9/python-hello
$ docker ps | grep hello
37ef88012ba8        ydd9/python-hello
             "python ./app.py"        42 seconds ago      Up 41 seconds       8080/tcp, 0.0.0.0:8888->8081/tcp   elastic_fermat
```

If now you want all colleagues access your website, you switch to use Bridge network settings in VirtualBox
You should then `$ docker run --network=host ydd9/python-hello`, next is to find Debian VM IP address 192.168.0.39 via `$ ip a` So now from your host win10, your guest Debian and your colleagues system, open website `192.168.0.39:8080` should display the "Hello, world!"  https://forums.docker.com/t/how-to-access-docker-container-from-another-machine-on-local-network/4737/11


# Push Docker image to dockerhub for easy download and reuse
https://docs.docker.com/docker-cloud/builds/push-images/
```
$ export DOCKER_ID_USER="username"
$ docker login
$ docker tag my_image $DOCKER_ID_USER/my_image
$ docker push $DOCKER_ID_USER/my_image
# verify in docker hub
```


# (Optional) Save Docker image locally and import for use
```
$ docker save my_image > my_image.tar

$ docker load --input my_image.tar
```

**Deploy this app with Kubernetes(K8s)**

# Use kubeadm to setup Kubernetes cluster
Official guid is very clear, but you may encounter many different problems, you can check problems I had via [this link](https://github.com/YDD9/YDD9.github.io/blob/master/_posts/2017-12-21-Docker-Kubernetes.md#kubeadm-install)

You can also deploy K8s cluster manually, this hard way is good for people who want to understand all and
admin the process. https://github.com/kelseyhightower/kubernetes-the-hard-way or the official guide.


# Prepare deployment.yml and deploy app with kubectl
template found https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/
```
$ kubectl apply -f https://github.com/YDD9/docker-app-hello/blob/master/deployment.yml
error: yaml: line 2: mapping values are not allowed in this context
# this githube link is not able to give a pure .yml file, but you click button raw, then the link should work
# https://raw.githubusercontent.com/YDD9/docker-app-hello/master/deployment.yml

# if don't have github, you can put the file on cluster master locally
$ kubectl apply -f /media/share/deployment.yml
deployment "python-hello-deployment" created
```

# Check deployment status and troubleshot
### Once deployment created, check the process status
```
$ kubectl describe deployment python-hello-deployment
Name:                   python-hello-deployment
Namespace:              default
CreationTimestamp:      Sat, 30 Dec 2017 12:15:46 -0500
Labels:                 app=python-hello
Annotations:            deployment.kubernetes.io/revision=1
                        kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"apps/v1beta2","kind":"Deployment","metadata":{"annotations":{},"name":"python-hello-deployment","namespace":"default"},"spec":{"replicas...
Selector:               app=python-hello
Replicas:               2 desired | 2 updated | 2 total | 0 available | 2 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=python-hello
  Containers:
   python-hello:
    Image:        ydd9/python-hello
    Port:         8080/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      False   MinimumReplicasUnavailable
  Progressing    False   ProgressDeadlineExceeded
OldReplicaSets:  <none>
NewReplicaSet:   python-hello-deployment-6b69b45664 (2/2 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  12m   deployment-controller  Scaled up replica set python-hello-deployment-6b69b45664 to 2
```

### Why deployment is unavailable, check deployed pods
```
$ kubectl describe pods python-hello-deployment
Name:           python-hello-deployment-6b69b45664-54898
Namespace:      default
Node:           kubemaster/192.168.0.39
Start Time:     Sat, 30 Dec 2017 12:15:47 -0500
Labels:         app=python-hello
                pod-template-hash=2625601220
Annotations:    <none>
Status:         Pending
IP:
Controlled By:  ReplicaSet/python-hello-deployment-6b69b45664
Containers:
  python-hello:
    Container ID:
    Image:          ydd9/python-hello
    Image ID:
    Port:           8080/TCP
    State:          Waiting
      Reason:       ContainerCreating
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-jtvxk (ro)
Conditions:
  Type           Status
  Initialized    True
  Ready          False
  PodScheduled   True
Volumes:
  default-token-jtvxk:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-jtvxk
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason                  Age                 From                 Message
  ----     ------                  ----                ----                 -------
  Normal   Scheduled               13m                 default-scheduler    Successfully assigned python-hello-deployment-6b69b45664-54898 to kubemaster
  Normal   SuccessfulMountVolume   13m                 kubelet, kubemaster  MountVolume.SetUp succeeded for volume "default-token-jtvxk"
  Warning  FailedCreatePodSandBox  13m (x12 over 13m)  kubelet, kubemaster  Failed create pod sandbox.
  Normal   SandboxChanged          3m (x531 over 13m)  kubelet, kubemaster  Pod sandbox changed, it will be killed and re-created.

Name:           python-hello-deployment-6b69b45664-vqj57
Namespace:      default
Node:           kubemaster/192.168.0.39
Start Time:     Sat, 30 Dec 2017 12:15:47 -0500
Labels:         app=python-hello
                pod-template-hash=2625601220
Annotations:    <none>
Status:         Pending
IP:
Controlled By:  ReplicaSet/python-hello-deployment-6b69b45664
Containers:
  python-hello:
    Container ID:
    Image:          ydd9/python-hello
    Image ID:
    Port:           8080/TCP
    State:          Waiting
      Reason:       ContainerCreating
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-jtvxk (ro)
Conditions:
  Type           Status
  Initialized    True
  Ready          False
  PodScheduled   True
Volumes:
  default-token-jtvxk:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-jtvxk
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason                  Age                 From                 Message
  ----     ------                  ----                ----                 -------
  Normal   Scheduled               13m                 default-scheduler    Successfully assigned python-hello-deployment-6b69b45664-vqj57 to kubemaster
  Normal   SuccessfulMountVolume   13m                 kubelet, kubemaster  MountVolume.SetUp succeeded for volume "default-token-jtvxk"
  Warning  FailedCreatePodSandBox  8m (x272 over 13m)  kubelet, kubemaster  Failed create pod sandbox.
  Normal   SandboxChanged          3m (x535 over 13m)  kubelet, kubemaster  Pod sandbox changed, it will be killed and re-created.
```

### Why FailedCreatePodSandBox
```
$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                       READY     STATUS              RESTARTS   AGE
default       python-hello-deployment-6b69b45664-54898   0/1       ContainerCreating   0          15m
default       python-hello-deployment-6b69b45664-vqj57   0/1       ContainerCreating   0          15m
kube-system   etcd-kubeslave1                            1/1       Running             0          33m
kube-system   kube-apiserver-kubeslave1                  1/1       Running             0          33m
kube-system   kube-controller-manager-kubeslave1         1/1       Running             0          33m
kube-system   kube-dns-6f4fd4bdf-sxhhd                   0/3       ContainerCreating   0          34m
kube-system   kube-flannel-ds-b67hx                      1/2       CrashLoopBackOff    11         33m
kube-system   kube-flannel-ds-v9dp5                      1/2       CrashLoopBackOff    8          20m
kube-system   kube-proxy-4q2pw                           1/1       Running             0          20m
kube-system   kube-proxy-9nfq7                           1/1       Running             0          34m
kube-system   kube-scheduler-kubeslave1                  1/1       Running             0          33m
```

### Why kube-flannel CrashLoopBackOff and kube-dns keep ContainerCreating...
https://github.com/kubernetes/kubeadm/issues/578
check journalctl on master and on nodes
check kubelet status on master and on nodes
check `kubectl describe pods python-hello-deployment`

work on each errors, normally the kube-dns is troublesome, try different CNI is last option
```
$ journalctl -u kubelet
$ journalctl -xe

$ systemctl status kubelet -l
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
  Drop-In: /etc/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since Sat 2017-12-30 17:32:35 EST; 45min ago
     Docs: http://kubernetes.io/docs/
 Main PID: 1337 (kubelet)
    Tasks: 18 (limit: 4915)
   Memory: 53.6M
      CPU: 1min 39.860s
   CGroup: /system.slice/kubelet.service
           └─1337 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/ku

Dec 30 18:10:04 kubeslave1 kubelet[1337]: W1230 18:10:04.407499    1337 reflector.go:341] k8s.io/kubernetes/pkg/kubelet/config/aDec 30 18:15:41 kubeslave1 kubelet[1337]: W1230 18:15:41.698709    1337 raw.go:87] Error while processing event ("/sys/fs/cgroupDec 30 18:15:41 kubeslave1 kubelet[1337]: W1230 18:15:41.703225    1337 raw.go:87] Error while processing event ("/sys/fs/cgroupDec 30 18:15:41 kubeslave1 kubelet[1337]: W1230 18:15:41.703284    1337 raw.go:87] Error while processing event ("/sys/fs/cgroupDec 30 18:15:41 kubeslave1 kubelet[1337]: W1230 18:15:41.703315    1337 raw.go:87] Error while processing event
```
Further kube-dns crash issues solutions
https://github.com/kubernetes/kubernetes/issues/54910

Check `cat /var/log/message` log.
Also check kubelet log via `journalctl -u kubelet` for now.
Service status check `systemctl status kubelet -l`

Check pod info `kubectl logs --namespace=kube-system $(kubectl get pods --namespace=kube-system -l k8s-app=kube-dns -o name) -c kubedns`
https://github.com/coreos/coreos-kubernetes/issues/878
```
E0211 20:21:19.857061       1 reflector.go:201] k8s.io/dns/pkg/dns/dns.go:147: Failed to list *v1.Endpoints: Get https://10.96.0.1:443/api/v1/endpoints?resourceVersion=0: dial tcp 10.96.0.1:443: getsockopt: no route to host
```
Check there is no firewall issue between Master Node and Worker Node

Check iptable `iptables -S` https://www.digitalocean.com/community/tutorials/how-to-list-and-delete-iptables-firewall-rules

try turning off firewall and drop rule which may close connectivity

But best solution would be on basis of error which you are getting, so check deamon log at /var/lib/messeges and systemctl status kubelet -l

For Kube dns mostly issue are kubelet and docker not in sync i.e. check kubelet and docker using same cgroup driver

Issue : kube-dns crashloopbackoff after flannel/weave install
Solution:
We can not use `--cgroup-driver=systemd`, it must use `cgroupfs`!
Which is different from official described by [Docker installation for k8s](https://kubernetes.io/docs/setup/independent/install-kubeadm/)

Docker runtime cgroups setups via `sudo dockerd --exec-opt native.cgroupdriver=cgroupfs` or
Set the docker daemon to use cgroupfs:
```
cat << EOF > /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=cgroupfs"]
}
EOF

# Set kubelet use cgroupfs too:
$ vi /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
# Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=systemd"
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs"

# !!! Reload daemon setting then restart kubelet and docker EVERY TIME FILE CHANGE !!!
systemctl daemon-reload && systemctl restart kubelet docker
# verify
docker info |grep -i cgroup
```
Important NOTE
you'll need to change cgroup driver also in your nodes.


healthz error
```
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10255/healthz' failed with error: Get http://localhost:10255/healthz: dial tcp [::1]:10255: getsockopt: connection refused.
```
Check `journalctl -ex | grep kubelet --color`
```
Feb 11 16:53:30 master0 kubelet[25759]: error: failed to run Kubelet: failed to create kubelet: misconfiguration: kubelet cgroup driver: "cgroupfs" is different from docker cgroup driver: "systemd"
Feb 11 16:53:30 master0 systemd[1]: kubelet.service: Main process exited, code=exited, status=1/FAILURE
Feb 11 16:53:30 master0 systemd[1]: kubelet.service: Unit entered failed state.
Feb 11 16:53:30 master0 systemd[1]: kubelet.service: Failed with result 'exit-code'.
```

swap must not be used.
https://serverfault.com/questions/684771/best-way-to-disable-swap-in-linux

1. run swapoff -a : this will immediately disable swap.
2. remove/comment any swap entry from /etc/fstab.
3. reboot the system. If the swap is gone, good. If, for some reason, it is still here, you had to remove the swap partition. Repeat steps 1 and 2 and, after that, use fdisk or parted to remove the (now unused) swap partition. ...
4. reboot.

Most probably not relevant here but keep an eye: https://github.com/kubernetes/kubernetes/pull/50377
add flag --fail-swap-on=false to .../hack/local-up-cluster.sh

# kube-dns still failing and needs to be deleted
Before I used Flannel CNI network plugin, now I `kubeadm reset` and use Calico the same issue.
```
kubectl get pods --all-namespaces
NAMESPACE     NAME                                     READY     STATUS    RESTARTS   AGE
kube-system   calico-etcd-tv9mk                        1/1       Running   0          22m
kube-system   calico-kube-controllers-d6c6b9b8-zvk9k   1/1       Running   0          22m
kube-system   calico-node-d4lzq                        2/2       Running   0          22m
kube-system   etcd-kubeslave1                          1/1       Running   0          23m
kube-system   kube-apiserver-kubeslave1                1/1       Running   0          23m
kube-system   kube-controller-manager-kubeslave1       1/1       Running   0          24m
kube-system   kube-dns-6f4fd4bdf-xb9xr                 0/3       Pending   0          24m
...

# https://stackoverflow.com/questions/38263252/how-to-delete-kubernetes-pods-and-other-resources-in-the-system-namespace
# delete a kube-dns pod from kube-system
$ kubectl get deployment --namespace=kube-system
NAME                       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
calico-kube-controllers    1         1         1            1           24m
calico-policy-controller   0         0         0            0           24m
kube-dns                   1         1         1            0           26m

$ kubectl delete deployment kube-dns --namespace=kube-system
deployment "kube-dns" deleted

# deploy again the kube-dns
$ kubectl apply -f https://raw.githubusercontent.com/kelseyhightower/kubernetes-the-hard-way/master/deployments/kube-dns.yaml
```

```
# delete a node from cluster
$ kubectl delete node <node_name1> <node_name2>
```

### retry with Flannel with hard coded IP range 10.244.0.0/16 and kube-dns works
Now the issue is one step further, but my newly added node never Ready, so I think my image is really
not good, so I clone the master image and make sure new MAC, new IP to generate a new VM, and everything
works now
```
root@node40$ kubeadm init --pod-network-cidr=10.244.0.0/16

root@node40:~# kubectl get pods --all-namespaces
NAMESPACE     NAME                             READY     STATUS    RESTARTS   AGE
kube-system   etcd-node40                      1/1       Running   0          10m
kube-system   kube-apiserver-node40            1/1       Running   0          10m
kube-system   kube-controller-manager-node40   1/1       Running   0          10m
kube-system   kube-dns-6f4fd4bdf-b6bgb         3/3       Running   0          11m
kube-system   kube-flannel-ds-9h5vg            1/1       Running   0          9m
kube-system   kube-flannel-ds-c6gvc            1/1       Running   0          9m
kube-system   kube-proxy-j4bzd                 1/1       Running   0          11m
kube-system   kube-proxy-qgh7c                 1/1       Running   0          10m
kube-system   kube-scheduler-node40            1/1       Running   0          10m

root@node40:~# kubectl get nodes
NAME      STATUS    ROLES     AGE       VERSION
node40    Ready     master    12m       v1.9.0
node42    Ready     <none>    10m       v1.9.0
```

```
root@node42$ kubeadm join --token 962fce.ef1baf1602f7e1ab 192.168.0.40:6443 --discovery-token-ca-cert-hash sha256:d552ce6f4ac8523fc76f7bf24b244ea2a51b7d70188d1a566babc4cd23e8d65c
```

# Kubectl Autocomplete
```
$ apt-get install bash-completion
$ source <(kubectl completion bash) # setup autocomplete in bash, bash-completion package should be installed first.
```

# Deploy again the app
```
$ kubectl apply -f https://raw.githubusercontent.com/YDD9/docker-app-hello/master/deployment.yml

$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                       READY     STATUS              RESTARTS   AGE
default       python-hello-deployment-6b69b45664-744h8   0/1       ContainerCreating   0          6m
default       python-hello-deployment-6b69b45664-nqxw4   0/1       ContainerCreating   0          6m
kube-system   etcd-node40                                1/1       Running             0          1h
kube-system   kube-apiserver-node40                      1/1       Running             0          1h
kube-system   kube-controller-manager-node40             1/1       Running             0          1h
kube-system   kube-dns-6f4fd4bdf-b6bgb                   3/3       Running             0          1h
kube-system   kube-flannel-ds-9h5vg                      1/1       Running             0          1h
kube-system   kube-flannel-ds-c6gvc                      1/1       Running             0          1h
kube-system   kube-proxy-j4bzd                           1/1       Running             0          1h
kube-system   kube-proxy-qgh7c                           1/1       Running             0          1h
kube-system   kube-scheduler-node40                      1/1       Running             0          1h

# switch to work node
$ journalctl -ex
...
node42 kernel: IPv6: ADDRCONF(NETDEV_CHANGE): eth0: link becomes ready
node42 kernel: cni0: port 1(veth8f9db634) entered blocking state
node42 kernel: cni0: port 1(veth8f9db634) entered disabled state
node42 kernel: device veth8f9db634 entered promiscuous mode
node42 kernel: cni0: port 1(veth8f9db634) entered blocking state
node42 kernel: cni0: port 1(veth8f9db634) entered forwarding state
node42 systemd-udevd[32394]: Could not generate persistent MAC address for veth8f9db634: No such file or directory
node42 kubelet[12356]: E1231 16:29:55.333809   12356 cni.go:259] Error adding network: failed to set bridge addr: "cni0" already
node42 kubelet[12356]: E1231 16:29:55.334064   12356 cni.go:227] Error while adding to cni network: failed to set bridge addr: "
node42 kernel: cni0: port 1(veth0e544932) entered blocking state
...
```

### To workaround the Linux bug persistent MAC address on VirtualBox https://github.com/systemd/systemd/issues/3374
file exmaple https://serverfault.com/questions/837454/interface-will-not-rename-under-systemd
```
# cp /usr/lib/systemd/network/99-default.link /etc/systemd/network/99-default.link
# replace MACAddressPolicy=persistent with MACAddressPolicy=none

# user tested with (systemd 232, Debian 9 stretch):
# nano /etc/systemd/network/99-default.link
[Link]
NamePolicy=kernel database onboard slot path
MACAddressPolicy=none
```


### Error adding network: failed to set bridge cni
This issue happens when you used `kubeadm reset`, but cni is not cleaned up
You have to manually clean up and re-create cluster again.
https://github.com/kubernetes/kubernetes/issues/39557
https://stackoverflow.com/questions/41359224/kubernetes-failed-to-setup-network-for-pod-after-executed-kubeadm-reset

```
kubeadm reset

systemctl stop kubelet
systemctl stop docker

rm -rf /var/lib/cni/
rm -rf /var/lib/kubelet/*
rm -rf /etc/cni/
rm -rf /run/flannel
rm -rf /run/calico


# delete network interface in case of centOS ifconfig cni0 down
ip link delete cni0
ip link delete flannel.1
ip link delete docker0

systemctl start docker
```
In HAcluster with seperate nodes for etcd cluster, the flannel config will be retrieved directly there, so delete above won't delete flannel either!!!

Repeat the cluster setup and app deployment works
```
root@node40:~/Downloads# kubectl get pods --all-namespaces
NAMESPACE     NAME                                       READY     STATUS    RESTARTS   AGE
default       python-hello-deployment-6b69b45664-j4768   1/1       Running   0          2m
default       python-hello-deployment-6b69b45664-kwnlz   1/1       Running   0          2m
kube-system   etcd-node40                                1/1       Running   0          3m
kube-system   kube-apiserver-node40                      1/1       Running   0          3m
kube-system   kube-controller-manager-node40             1/1       Running   0          3m
kube-system   kube-dns-6f4fd4bdf-zvhmn                   3/3       Running   0          4m
kube-system   kube-flannel-ds-ddmd8                      1/1       Running   0          3m
kube-system   kube-flannel-ds-phprc                      1/1       Running   0          4m
kube-system   kube-proxy-6qs4m                           1/1       Running   0          4m
kube-system   kube-proxy-qmnhz                           1/1       Running   0          3m
kube-system   kube-scheduler-node40                      1/1       Running   0          4m
```

# Further info check other *.md files
