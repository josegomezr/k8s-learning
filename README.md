K8S Learning Ground!
===

This is gonna be my journal of learning k8s!

TOC
---

* [🚨 BEFORE YOU BEGIN 🚨](#-before-you-begin-)
* FIRST DAY
* [How to read this?](#how-to-read-this)
* [TL;DR I want easy route](#tldr-i-want-easy-route)
* [⚠️ WARNING ⚠️](#%EF%B8%8F-warning-%EF%B8%8F)
* [What should I do first? NAMESPACES!](#what-should-i-do-first-namespaces)
* [Have something running, even if you can't access it yet](#have-something-running-even-if-you-cant-access-it-yet)
* [Connecting your deployment to the "outside world"](#connecting-your-deployment-to-the-outside-world)
* [Using request-rewriting! [optional]](#using-request-rewriting-optional)
* SECOND DAY
* [2nd session](#2nd-session)
* [Adapting our pods to be python http.server](#adapting-our-pods-to-be-python-httpserver)
* [Creating the volume](#creating-the-volume)
* [Proving the storage is persistent](#proving-the-storage-is-persistent)"

🚨 BEFORE YOU BEGIN 🚨
---

1. GET! SOME! K8s! (Minikube, k3s, anything you can run `kubectl` against will do.).

How to read this?
---

I'll help myself and whoever reads this not to do the classic "<s>Go to the YAML files and figure stuff out like the real hackers!</s>"

K8s is a VERY big beast, with a TON of moving parts. I'm not focused on learning how the cluster WORKS inside, I'd also like to learn German and my brain power is not infinite, so I'd like to get something more practical. I'll learn how to use it to deploy stuff (namely, how to structure/write YAML files that `kubectl` can use).

I'll do my best on keeping the files annotated so you (and future me) can follow through what's what.

TL;DR I want easy route
---

```
# create everything
kubectl apply -f .
# destroy everything
kubectl delete -f .
```

It'll create (or destroy) all resources, read below 👇 what do they do and mean.

⚠️ **WARNING** ⚠️
---

Whatever that you read CAN and PROBABLY WILL be inaccurate, I suck at reading docs, so mostly I just throw 9999 commands and then take notes on what worked. ALMOST EVERYTHING HERE is MY black-box presumption on what it is or happening.

_💬 yes, httpbin is my test application, too lazy to create one from scratch 😅_

**📝 IMPORTANT NOTE: for my brain sake, I've suffixed resource names, so they're much easily tracked through the source files.**

What should I do first? NAMESPACES!
---

1️⃣ let's go with Namespaces, that one is defined in [httpbin/httpbin.yml][httpbin-namespace-definition-file]. In K8s **namespaces isolate resources so they don't clash on each other**.

Lets imagine this Namespace as a tiny universe where our workload will be living.

**When you create a namespace, absolutely NOTHING** (besides the object being created) **happens**. Literally feels like you did NOTHING.

So **every resource** we'll use here, **WILL BE inside our** newly created **namespace**.

* 📝 **NOTE:** have a look at the metadata section, all K8S objects have that, it behaves like a `<head>` tag in a browser.
* 📝 **NOTE:** all K8S objects will need a `name`.
* 📝 **NOTE:** the labels section is not mandatory, but it can be used to filter resources, it doesn't hurt I guess 🤷.

To create the namespace, you can apply (💡 ahá, `apply` is a subcommand of `kubectl` that will create/update resources) the `httpbin/httpbin.yml` file alone like:

```
$ kubectl apply -f httpbin/httpbin.yml
namespace/httpbin-ns created
```

📝 **NOTE:** K3S had no issue, but rancher/longhorn replied with a message saying that the namespace was missing an annotation, but it also said it auto patched it? So I guess it solves itself.

As I said earlier, **this has** absolutely **NO visible effect**, other than we'll need `--namespace httpbin` to fetch resources. Like:

```
$ kubectl --namespace httpbin-ns get all
```

WHICH `kubectl` will say that we have nothing, well, because there's nothing!

Have something running, even if you can't access it yet
---

2️⃣ Now let's spin up some workload

_💬 Workload in K8S seems to be a slang for Container/Process/Application anything that creates work in the cluster_

The smallest "unit of work" in K8S is a `pod`, a `pod` is a set of containers, and as said earlier it has A SHIT TON of complexity behind that I completely ignore :D. However, I do know that:

* Although you can create a `pod` with kubectl like you were to do `docker run` (they call this a "naked pod") it's discouraged in the documentation (section [configuration best practices][config-best-practices-naked-post]). A "naked pod" is a pod not tied to a ReplicaSet or a Deployment, we'll see them later. 

* A deployment will create underneath a ReplicaSet (which will be making sure we have those pods running no-matter-what) and the previously mentioned pods (well, what the ReplicaSet controls).

_* 💬 AFAIU there seems to be a "common pattern" to have 1 pod -> 1 container. But this is not set in stone, yet?_

Let's go the deployment route, reading naked in a documentation doesn't look correct, and the file definition doesn't seem to difficult either. You can see the deployment definition [httpbin/httpbin_deployment.yml][httpbin-deployment-definition-file].

* 📝 **NOTE:** see that the **namespace name** matches the one we created on [👉1️⃣](#what-should-i-do-first-namespaces)
* 📝 **NOTE:** `spec.selector.matchLabels` has to have labels that matches (the name implies it, duh) what's in `spec.template.metadata.labels`. This will be used by the ReplicaSet created under the hood.
  + 
* 📝 **NOTE:** `httpbin` exposes port 80, mind the `ports.containerPort` inside each container block.

Applying this file alone:

```
$ kubectl apply -f httpbin/httpbin_deployment.yml 
deployment.apps/httpbin-deployment created
```

With this done, now we can fetch resources like we did before.

* **📝 NOTE: `--namespace` flag is required**

```
$ kubectl --namespace httpbin-ns get all
NAME                                      READY   STATUS    RESTARTS   AGE
pod/httpbin-deployment-6754d5d56f-5mfjn   1/1     Running   0          56s
pod/httpbin-deployment-6754d5d56f-222sn   1/1     Running   0          56s

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/httpbin-deployment   2/2     2            2           56s

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/httpbin-deployment-6754d5d56f   2         2         2       56s
```
So, a number of pods were created, exactly `spec.replicas` amount of pods. This new resources are now living inside the cluster, but they're not yet accessible (we can say Achievement Unlocked?) but let's try to reach one node from within the cluster. First, let's see what they look like.

* **📝 NOTE (AGAIN): `--namespace` flag**

```
$ kubectl --namespace httpbin-ns get pods -o wide
NAME                                  READY   STATUS    RESTARTS   AGE    IP           NODE        NOMINATED NODE   READINESS GATES
httpbin-deployment-6754d5d56f-5mfjn   1/1     Running   0          7m5s   10.42.0.80   localhost   <none>           <none>
httpbin-deployment-6754d5d56f-222sn   1/1     Running   0          7m5s   10.42.0.79   localhost   <none>           <none>
```

Ok, so there are some IP addresses, since they don't match my network I'll assume they're internal IP's within the cluster, I think the best idea here will be creating a pod alongside those 3, since they're in the same namespace, they should be able to communicate with each other. Considering that `httpbin` listens by default to `0.0.0.0:80`.

I'll take the first one `10.42.0.80` and spin a curl image so we throw an HTTP request to it.

* 📝 **NOTE:** `--namespace` flag is **REALLY** important here
* 📝 **NOTE:** order matters, for whatever reason, `--rm` won't work if it's at first, putting it after `--image` worked 🤷

```
kubectl --namespace httpbin-ns run -i --tty --image curlimages/curl --rm curl sh
```

It might take a bit since it's pulling the image, but you should get a working sh session. Now inside of it, let's try to reach the pod we targeted

```
# inside our curl container.
$ curl 10.42.0.80/get
{
  "args": {}, 
  "headers": {
    "Accept": "*/*", 
    "Host": "10.42.0.80", 
    "User-Agent": "curl/7.75.0-DEV"
  }, 
  "origin": "10.42.0.84", 
  "url": "http://10.42.0.80/get"
}
```

✨✨✨

**GREAT! We have:**
* ✅ pods working!
* ✅ they're reachable from within the cluster!

✨✨✨

Now we need some sort of way of **exposing** those containers to the **outside world**.

Connecting your deployment to the "outside world"
---

_💬 AFAIU it's debatable if it's really the "ouside world" but it's not within the cluster, that's for sure._

3️⃣ Next thing to do is: throw a service. **A Service will allow our pods to be exposed**. There are 2 main types of services:

* `NodePort`: as it's looks, it'll give you a port (in the cluster) that will be routed to the deployments.
* `LoadBalancer`: this one, according to the docs, exposes your deployment using a cloud provider load balancer. I guess this is more similar to an AWS/GCP \*LB. I'll be using this one, although I don't quite sure get the difference between the two of them in my local K3S. Later I'll do tests in AWS.

Check out the service definition [httpbin/httpbin_service.yml][httpbin-service-definition-file].

* 📝 **NOTE:** see that the **namespace name** matches the one we created on [👉1️⃣](#what-should-i-do-first-namespaces)
* 📝 **NOTE:** the service behaves as an intermediary "container"/"pod" between the outside world and your pods. It'll link traffic from the service port (`spec.ports[0].port`) into the `pod` port (`spec.ports[0].targetPort`).
* 📝 **NOTE:** mind `spec.ports[0].targetPort` **has to match** `ports.containerPort` inside each container block.
* 📝 **NOTE:** `spec.selector.app` **has to match** whatever we wrote in the **[deployment definition file][httpbin-deployment-definition-file] on key** `spec.template.metadata.labels`.

Let's create it now:

```
$ kubectl apply -f httpbin/httpbin_service.yml
```

If everything goes well, we get to have

```
service/httpbin-load-balancer created
```

Now let's inspect what was created:

```
kubectl --namespace httpbin-ns describe service
Name:                     httpbin-load-balancer
Namespace:                httpbin-ns
Labels:                   target=httpbin
Annotations:              <none>
Selector:                 app=httpbin-app
Type:                     LoadBalancer
IP Families:              <none>
IP:                       10.43.59.81
IPs:                      10.43.59.81
Port:                     http  80/TCP
TargetPort:               80/TCP
NodePort:                 http  30275/TCP
Endpoints:                10.42.0.89:80,10.42.0.90:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

So according to this, if I throw a request to `IP` I should get something:

```
# in my machine
$ curl --dump-header - 192.168.122.76/get
HTTP/1.1 404 Not Found
Content-Type: text/plain; charset=utf-8
Vary: Accept-Encoding
X-Content-Type-Options: nosniff
Date: Wed, 17 Feb 2021 20:01:27 GMT
Content-Length: 19

404 page not found
```

Hmmm, `httpbin` by default writes `Server: gunicorn/$version`, this doesn't have that. So we're not hitting our pod yet.


Let's use that `NodePort` :

```
# in my machine
$ curl --dump-header - 192.168.122.76:30275/get
HTTP/1.1 200 OK
Server: gunicorn/19.9.0
Date: Wed, 17 Feb 2021 20:04:04 GMT
Connection: keep-alive
Content-Type: application/json
Content-Length: 198
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true

{
  "args": {}, 
  "headers": {
    "Accept": "*/*", 
    "Host": "192.168.122.76:8000", 
    "User-Agent": "curl/7.66.0"
  }, 
  "origin": "10.42.0.1", 
  "url": "http://192.168.122.76:8000/get"
}
```

✨✨✨

**GREAT NEWS**
* ✅ Our service is reachable from the outside of the cluster!

✨✨✨

Using request-rewriting! [optional]
---

This is a classic thing when having a reverse proxy, we can have a (let's say) nginx taking requests, and routing them to specific upstreams (nginx jargon for an app listening in some socket). The main benefit of this is that you can have a lot of services running in the same host, and nginx will take the job of appropriately route them (based on pathname, host, or whatever you config).

Maybe in today's world is not so much used? But I find it useful, so lets see it.

So, K8S has a default http server running, that apparently I can exploit using an **Ingress**. **An ingress tells the cluster to**, well, **_ingress_ traffic to a service**.

_💬 I'll base this heavily on what I have at hand which is K3S. in my K3S cluster `traefik` is used instead of `nginx` (as the official K8S docs show). This will affect some metadata in the ingress definition._

Let's see it [here in the ingress definition file httpbin/httpbin_ingress.yml][httpbin-ingress-definition-file]

* 📝 **NOTE:** the full docs for [traefik annotations is here][httpbin-traefik-k8s-annotations]
* 📝 **NOTE:** **SPECIAL ATTENTION** to `spec.rules[0].http.paths[0].backend.service.name` **gotta match** the load-balancer name!
* 📝 **NOTE:** **SPECIAL ATTENTION** to `spec.rules[0].http.paths[0].backend.service.port.number` **gotta match** the load-balancer exposed port! _(💡 AHÁ! here's used! 🤩)_

Let's apply this!

```
$ kubectl apply -f httpbin/httpbin_ingress.yml 
ingress.networking.k8s.io/httpbin-ingress created
```

Now let's inspect it:

```
$ kubectl --namespace httpbin-ns describe ingress
Name:             httpbin-ingress
Namespace:        httpbin-ns
Address:          192.168.122.76
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>) # no idea why this error happens
Rules:
  Host        Path  Backends
  ----        ----  --------
  *           
              /abc   httpbin-load-balancer:8000 (10.42.0.79:80,10.42.0.80:80)
Annotations:  traefik.ingress.kubernetes.io/rule-type: PathPrefixStrip
Events:       <none>
```

So according to this, if I throw a request to `$CLUSTER_IP/abc/get` it should reply as before, let's see!

```
$ curl --dump-header - 192.168.122.76/abc/get
HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Content-Length: 353
Content-Type: application/json
Date: Wed, 17 Feb 2021 20:18:50 GMT
Server: gunicorn/19.9.0
Vary: Accept-Encoding

{
  "args": {}, 
  "headers": {
    "Accept": "*/*", 
    "Accept-Encoding": "gzip", 
    "Host": "192.168.122.76", 
    "User-Agent": "curl/7.66.0", 
    "X-Forwarded-Host": "192.168.122.76", 
    "X-Forwarded-Prefix": "/abc", 
    "X-Forwarded-Server": "traefik-6f9cbd9bd4-rw8df"
  }, 
  "origin": "10.42.0.1", 
  "url": "http://192.168.122.76/get"
}
```

✨✨✨

**GREAT NEWS**
* ✅ YES! Now we can reach the service without a port! :D

✨✨✨

---

2nd session
===========

Ok, so far our pods works, but they:

1. Are not connected to any "persistent" service (like a DB or something).
2. Are ephimeral, anything they write to the container fs will be lost upon
redeployment and/or scaling.

The first item I feel it a bit complex for my introduction, so I'll go first
with the second, let's have some persistent storage, but `httpbin` wouldn't pay
attention to it 🤔 we gonna need another application.

Honestly, I'm not feeling like making yet another application, so to demonstrate
this I'll go the hyper lazy way. I'll spin up a pod with a python web server.

Practically mimicing this:

```
docker run --rm -it -p 8000:8000 python:3-alpine sh -c "mkdir -p /var/data; cd /var/data; echo $(hostname): $(date) > "$(hostname)-$(date +%s).txt"; python3 -m http.server"
```

**What this container does?**

1. 🆕 Creates `/var/data`
2. 🗂 Change dir to `/var/data`
3. ✍ Writes a file (conveniently, somewhat "unique" each time).
4. 🤖 Runs the server

So far this is exactly behaving as our pods so far, if I go into the container
and throw files to `/var/data` they'll be long gone after a new pod goes around.
We need some equivalent of `-v` flag in docker in our K8S setup.

For this, I'll create another folder and a new namespace so we don't clash with
our previous demo.

We'll call this namespace: `sample-server-ns`. Let's go, copy all in `httpbin`
into `sample_server`.

The lazy route would be, but you can do the exercise of manually creating everything:

```
mkdir -p sample_server/
cp httpbin/*.yml sample_server/
cd sample_server/
rename 'httpbin' 'sample_server' ./*
sed -i 's/httpbin/sample-server/' ./*.yml
```
* 📝 **NOTE:** don't know how available the `rename` command. I have it in all
my machines
* 📝 **NOTE:** this will mess up with the image name, but don't worry, we still need to change it down the line.
* 📝 **NOTE:** If you're cloning this repo, this is kinda pointless because the
files are already there fixed.

Adapting our pods to be python http.server
---

So, by default python3's `http.server` module will list all files in PWD/CWD and
listen in port **8000**. Let's add first the command and change the
`containerPort`/`targetPort`.

You can [see it here sample_server/sample_server_deployment.yml][sample_server-deployment-definition-file]. Here let's see the diff for
the deployment:

```diff
       containers:
# here the image changes
-      - image: docker.io/kennethreitz/httpbin
+      - image: 'python:3-alpine'
         imagePullPolicy: IfNotPresent
# adding the command
+        command: ['sh', '-c', 'mkdir -p /var/data; cd /var/data; echo $(hostname): $(date) > "$(hostname)-$(date +%s).txt"; python3 -m http.server']
         # expose the http service port of our image.
         ports:
# and here the containerPort
-        - containerPort: 80
+        - containerPort: 8000
```

For this example, we'll take out the ingress, we no gonna use it right now.

_💬 Later on (maybe another deployment?) we might take it back._

```
rm sample_server/sample_server_ingress.yml
```

And last let's also tweak the service. You can
[see it here sample_server/sample_server_service.yml][sample_server-service-definition-file]. Here let's see the diff for the service:

```
# port here.
# Funny enough, this port has to be unique if you're using a setup like me
# where you have a single node K8S. Else you'll endup having a waiting for
# schedule pod that will never start because of the busy port.
-    port: 8100
+    port: 8101
# target port here
-    targetPort: 80 # this is the port exposed in the pods.
#...
+    targetPort: 8000 # this is the port exposed in the pods.
```

Now, let's apply this and see how it behaves!

```
$ kubectl apply -f .
namespace/sample-server-ns unchanged
deployment.apps/sample-server-deployment configured
service/sample-server-load-balancer created
```

Now let's have a look at the describe block:

```
$ kubectl --namespace sample-server-ns describe svc
Name:                     sample-server-load-balancer
Namespace:                sample-server-ns
Labels:                   target=sample-server
Annotations:              <none>
Selector:                 app=sample-server-app
Type:                     LoadBalancer
IP Families:              <none>
IP:                       10.43.254.216
IPs:                      10.43.254.216
LoadBalancer Ingress:     192.168.122.76
Port:                     http  8101/TCP
TargetPort:               8000/TCP
NodePort:                 http  31008/TCP
Endpoints:                10.42.0.127:8000,10.42.0.128:8000
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

Now, let's query that node:

```
$ curl --dump-header - 192.168.122.76:31008
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.9.1
Date: Fri, 19 Feb 2021 14:53:57 GMT
Content-type: text/html; charset=utf-8
Content-Length: 571

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href="sample-server-deployment-6449668b9c-4wv7z-1613745329.txt">sample-server-deployment-6449668b9c-4wv7z-1613745329.txt</a></li>
<li><a href="sample-server-deployment-6449668b9c-lq48c-1613745329.txt">sample-server-deployment-6449668b9c-lq48c-1613745329.txt</a></li>
</ul>
<hr>
</body>
</html>

$ curl --dump-header - 192.168.122.76:32645
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.9.1
Date: Fri, 19 Feb 2021 12:26:00 GMT
Content-type: text/html; charset=utf-8
Content-Length: 434

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href="sample-server-deployment-69958fcd4c-qcnc6-1613737476.txt">sample-server-deployment-69958fcd4c-qcnc6-1613737476.txt</a></li>
</ul>
<hr>
</body>
</html>
```

Interesting! So we have now two different responses (which means our load balancer is load balancing! ✨) here:

```
sample-server-deployment-69958fcd4c-cdngw-1613737473.txt
sample-server-deployment-69958fcd4c-qcnc6-1613737476.txt
```

So, we need to mount some volume that can outlive our pods and it's mounted on
`/var/data` so those files can persist. If you don't believe me, well destroy it
and create it again. You'll see the filenames change :).

Creating the volume
-------------------

SO, not quite sure how to explain this. but adding volumes is not as simple as
it seems. In the docs you can see adding a `volumes` key in the `spec` block of
the deployment and everything magically works. But there's another required object
a bit difficult to understand. The `PersistentVolumeClaim`.

Quoting @fschueller, A `PersistentVolumeClaim`, tells K8S:

> “Please schedule this much from my volumes you know about so I can reference this in my pod spec”

So in our PVC (because acronyms are popz), we'll be telling K8S we want some
disk (120MiB in my case, pretty arbitrary number, apparently anything below
100MiB raises issues depending on the storage driver/controller/class? 🤷)

The file structure is rather simple, you can see it here [sample_server/sample_server_pv_claim.yml][sample_server-pv-claim-definition-file]

Let's apply it!

```
$ kubectl apply -f sample_server_pv_claim.yml
persistentvolumeclaim/sample-server-pv-claim created
```

Now if we inspect it:

```
# pv is short for persistentvolume
$ kubectl --namespace sample-server-ns describe pv sample-server-pv
Name:              sample-server-pv
Labels:            target=sample-server
Annotations:       <none>
Finalizers:        [kubernetes.io/pv-protection]
StorageClass:      local-storage
Status:            Available
Claim:             
Reclaim Policy:    Delete
Access Modes:      RWO
VolumeMode:        Filesystem
Capacity:          50Mi
Node Affinity:     
  Required Terms:  
    Term 0:        kubernetes.io/os in [linux]
Message:           
Source:
    Type:  LocalVolume (a persistent volume backed by local storage on a node)
    Path:  /var/pod_data
Events:    <none>
```

_💬 Weird it says 50Mi, but for my current use case I don't mind. Maybe is elastic? 🤔_

Now, we point in the deployment file, that we want to use this reserved space.

```
    spec:
# ... comments ...
+      volumes:
+      - name: sample-server-pv-storage # Achtung! this will be used below
+        persistentVolumeClaim:
+          claimName: sample-server-pv-claim # Achtung! sample_server_pv_claim.yml
# ... down below ...
        ports:
        - containerPort: 8000
+        volumeMounts:
+        - mountPath: "/var/data"
+          name: sample-server-pv-storage
```

As stated early, the file is already fixed in the repo, so we can apply it right
away. See the file [here sample_server/sample_server_deployment.yml][sample_server-deployment-definition-file]

```
$ kubectl apply -f sample_server_deployment.yml
deployment.apps/httpbin-deployment configured
```

So after we have this, let's sync up. I'll dump here my namespace, pod, services.

```
$ kubectl --namespace sample-server-ns describe pod
Name:         sample-server-deployment-6449668b9c-4wv7z
Namespace:    sample-server-ns
Priority:     0
Node:         localhost/192.168.122.76
Start Time:   Fri, 19 Feb 2021 15:35:27 +0100
Labels:       app=sample-server-app
              pod-template-hash=6449668b9c
Annotations:  <none>
Status:       Running
IP:           10.42.0.127
IPs:
  IP:           10.42.0.127
Controlled By:  ReplicaSet/sample-server-deployment-6449668b9c
Containers:
  sample-server-pod-container:
    Container ID:  containerd://2327a32ba6461457ab0849a0e599afe0bb613cf67c8a40dffb22b4b8ecd0d3a0
    Image:         python:3-alpine
    Image ID:      docker.io/library/python@sha256:e48dd378740e8dc680553ec91a2f8d80a87743a4ec43494695c21a2323a749ee
    Port:          8000/TCP
    Host Port:     0/TCP
    Command:
      sh
      -c
      mkdir -p /var/data; cd /var/data; echo $(hostname): $(date) > "$(hostname)-$(date +%s).txt"; python3 -m http.server
    State:          Running
      Started:      Fri, 19 Feb 2021 15:35:29 +0100
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/data from sample-server-pv-storage (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-x8ftm (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  sample-server-pv-storage:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  sample-server-pv-claim
    ReadOnly:   false
  default-token-x8ftm:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-x8ftm
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  26m   default-scheduler  Successfully assigned sample-server-ns/sample-server-deployment-6449668b9c-4wv7z to localhost
  Normal  Pulled     26m   kubelet            Container image "python:3-alpine" already present on machine
  Normal  Created    26m   kubelet            Created container sample-server-pod-container
  Normal  Started    26m   kubelet            Started container sample-server-pod-container


Name:         sample-server-deployment-6449668b9c-lq48c
Namespace:    sample-server-ns
Priority:     0
Node:         localhost/192.168.122.76
Start Time:   Fri, 19 Feb 2021 15:35:27 +0100
Labels:       app=sample-server-app
              pod-template-hash=6449668b9c
Annotations:  <none>
Status:       Running
IP:           10.42.0.128
IPs:
  IP:           10.42.0.128
Controlled By:  ReplicaSet/sample-server-deployment-6449668b9c
Containers:
  sample-server-pod-container:
    Container ID:  containerd://76c4083f9946060dc4e2ef73dfaafbe9d97d6943d045097e36ff9529014ab6b5
    Image:         python:3-alpine
    Image ID:      docker.io/library/python@sha256:e48dd378740e8dc680553ec91a2f8d80a87743a4ec43494695c21a2323a749ee
    Port:          8000/TCP
    Host Port:     0/TCP
    Command:
      sh
      -c
      mkdir -p /var/data; cd /var/data; echo $(hostname): $(date) > "$(hostname)-$(date +%s).txt"; python3 -m http.server
    State:          Running
      Started:      Fri, 19 Feb 2021 15:35:29 +0100
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/data from sample-server-pv-storage (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-x8ftm (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  sample-server-pv-storage:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  sample-server-pv-claim
    ReadOnly:   false
  default-token-x8ftm:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-x8ftm
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  26m   default-scheduler  running PreBind plugin "VolumeBinding": Operation cannot be fulfilled on persistentvolumeclaims "sample-server-pv-claim": the object has been modified; please apply your changes to the latest version and try again
  Normal   Scheduled         26m   default-scheduler  Successfully assigned sample-server-ns/sample-server-deployment-6449668b9c-lq48c to localhost
  Normal   Pulled            26m   kubelet            Container image "python:3-alpine" already present on machine
  Normal   Created           26m   kubelet            Created container sample-server-pod-container
  Normal   Started           26m   kubelet            Started container sample-server-pod-container


Name:         svclb-sample-server-load-balancer-8rxz5
Namespace:    sample-server-ns
Priority:     0
Node:         localhost/192.168.122.76
Start Time:   Fri, 19 Feb 2021 15:52:13 +0100
Labels:       app=svclb-sample-server-load-balancer
              controller-revision-hash=6b7bc6df8d
              pod-template-generation=2
              svccontroller.k3s.cattle.io/svcname=sample-server-load-balancer
Annotations:  <none>
Status:       Running
IP:           10.42.0.132
IPs:
  IP:           10.42.0.132
Controlled By:  DaemonSet/svclb-sample-server-load-balancer
Containers:
  lb-port-8101:
    Container ID:   containerd://49352eee37a75eea0e6d2b2bb750cb45ee488f89d12f768899de69c956002520
    Image:          rancher/klipper-lb:v0.1.2
    Image ID:       docker.io/rancher/klipper-lb@sha256:2fb97818f5d64096d635bc72501a6cb2c8b88d5d16bc031cf71b5b6460925e4a
    Port:           8101/TCP
    Host Port:      8101/TCP
    State:          Running
      Started:      Fri, 19 Feb 2021 15:52:15 +0100
    Ready:          True
    Restart Count:  0
    Environment:
      SRC_PORT:    8101
      DEST_PROTO:  TCP
      DEST_PORT:   8101
      DEST_IP:     10.43.254.216
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-x8ftm (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-x8ftm:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-x8ftm
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     CriticalAddonsOnly op=Exists
                 node-role.kubernetes.io/control-plane:NoSchedule op=Exists
                 node-role.kubernetes.io/master:NoSchedule op=Exists
                 node.kubernetes.io/disk-pressure:NoSchedule op=Exists
                 node.kubernetes.io/memory-pressure:NoSchedule op=Exists
                 node.kubernetes.io/not-ready:NoExecute op=Exists
                 node.kubernetes.io/pid-pressure:NoSchedule op=Exists
                 node.kubernetes.io/unreachable:NoExecute op=Exists
                 node.kubernetes.io/unschedulable:NoSchedule op=Exists
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  9m18s  default-scheduler  Successfully assigned sample-server-ns/svclb-sample-server-load-balancer-8rxz5 to localhost
  Normal  Pulled     9m17s  kubelet            Container image "rancher/klipper-lb:v0.1.2" already present on machine
  Normal  Created    9m16s  kubelet            Created container lb-port-8101
  Normal  Started    9m16s  kubelet            Started container lb-port-8101
```

And my service:

```
$ kubectl --namespace sample-server-ns describe svc
Name:                     sample-server-load-balancer
Namespace:                sample-server-ns
Labels:                   target=sample-server
Annotations:              <none>
Selector:                 app=sample-server-app
Type:                     LoadBalancer
IP Families:              <none>
IP:                       10.43.254.216
IPs:                      10.43.254.216
LoadBalancer Ingress:     192.168.122.76
Port:                     http  8101/TCP
TargetPort:               8000/TCP
NodePort:                 http  31008/TCP
Endpoints:                10.42.0.127:8000,10.42.0.128:8000
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

Finally, let's start querying. If everything goes well, we have to issue several
requests and always get the same response no matter what.


```
$ curl --dump-header - 192.168.122.76:31008
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.9.1
Date: Fri, 19 Feb 2021 15:03:29 GMT
Content-type: text/html; charset=utf-8
Content-Length: 571

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href="sample-server-deployment-6449668b9c-4wv7z-1613745329.txt">sample-server-deployment-6449668b9c-4wv7z-1613745329.txt</a></li>
<li><a href="sample-server-deployment-6449668b9c-lq48c-1613745329.txt">sample-server-deployment-6449668b9c-lq48c-1613745329.txt</a></li>
</ul>
<hr>
</body>
</html>

$ curl --dump-header - 192.168.122.76:31008
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.9.1
Date: Fri, 19 Feb 2021 15:03:30 GMT
Content-type: text/html; charset=utf-8
Content-Length: 571

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href="sample-server-deployment-6449668b9c-4wv7z-1613745329.txt">sample-server-deployment-6449668b9c-4wv7z-1613745329.txt</a></li>
<li><a href="sample-server-deployment-6449668b9c-lq48c-1613745329.txt">sample-server-deployment-6449668b9c-lq48c-1613745329.txt</a></li>
</ul>
<hr>
</body>
</html>

$ curl --dump-header - 192.168.122.76:31008
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.9.1
Date: Fri, 19 Feb 2021 15:03:31 GMT
Content-type: text/html; charset=utf-8
Content-Length: 571

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href="sample-server-deployment-6449668b9c-4wv7z-1613745329.txt">sample-server-deployment-6449668b9c-4wv7z-1613745329.txt</a></li>
<li><a href="sample-server-deployment-6449668b9c-lq48c-1613745329.txt">sample-server-deployment-6449668b9c-lq48c-1613745329.txt</a></li>
</ul>
<hr>
</body>
</html>

$ curl --dump-header - 192.168.122.76:31008
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.9.1
Date: Fri, 19 Feb 2021 15:03:31 GMT
Content-type: text/html; charset=utf-8
Content-Length: 571

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href="sample-server-deployment-6449668b9c-4wv7z-1613745329.txt">sample-server-deployment-6449668b9c-4wv7z-1613745329.txt</a></li>
<li><a href="sample-server-deployment-6449668b9c-lq48c-1613745329.txt">sample-server-deployment-6449668b9c-lq48c-1613745329.txt</a></li>
</ul>
<hr>
</body>
</html>

$ curl --dump-header - 192.168.122.76:31008
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.9.1
Date: Fri, 19 Feb 2021 15:03:32 GMT
Content-type: text/html; charset=utf-8
Content-Length: 571

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href="sample-server-deployment-6449668b9c-4wv7z-1613745329.txt">sample-server-deployment-6449668b9c-4wv7z-1613745329.txt</a></li>
<li><a href="sample-server-deployment-6449668b9c-lq48c-1613745329.txt">sample-server-deployment-6449668b9c-lq48c-1613745329.txt</a></li>
</ul>
<hr>
</body>
</html>
```

✨✨✨

**GREAT NEWS**
* ✅ Our deployment seems to have persistent storage! 🤩

✨✨✨

Proving the storage is persistent
---

To make sure we're REALLY outliving the pods with our storage, let's scale
the deployment down to 0 replicas, wait some 10 secs, then re-scale it to 3
nodes. If our setup is correct, we should see now 5 files in the list.

The previous 2, and 3 new ones, and the ALL have to keep replying the same list.
No matter how many times we request it.

To scale down a deployment:


**📝 NOTE:** `--namespace` flag
**📝 NOTE:** `--replicas` flag will edit the `spec.replicas` that we defiened as
`2` to `0` dynamically.

```
$ kubectl --namespace sample-server-ns scale deployment sample-server-deployment --replicas 0
deployment.apps/sample-server-deployment scaled
```

Let's see if we have pods

```
$ kubectl --namespace sample-server-ns get pods -o wide
deployment.apps/sample-server-deployment scaled
NAME                                        READY   STATUS        RESTARTS   AGE   IP            NODE        NOMINATED NODE   READINESS GATES
svclb-sample-server-load-balancer-8rxz5     1/1     Running       0          15m   10.42.0.132   localhost   <none>           <none>
sample-server-deployment-6449668b9c-lq48c   0/1     Terminating   0          32m   10.42.0.128   localhost   <none>           <none>
sample-server-deployment-6449668b9c-4wv7z   0/1     Terminating   0          32m   10.42.0.127   localhost   <none>           <none>
```

Ahá, terminating, let's wait a bit.

```
$ kubectl --namespace sample-server-ns get pods -o wide
NAME                                      READY   STATUS    RESTARTS   AGE   IP            NODE        NOMINATED NODE   READINESS GATES
svclb-sample-server-load-balancer-8rxz5   1/1     Running   0          16m   10.42.0.132   localhost   <none>           <none>
```

Great, no more pods, now let's scale it up!

```
$ kubectl --namespace sample-server-ns scale deployment sample-server-deployment --replicas 3
deployment.apps/sample-server-deployment scaled
```


Let's have a look at the pods:
```
$ kubectl --namespace sample-server-ns get pods -o wide
NAME                                        READY   STATUS              RESTARTS   AGE   IP            NODE        NOMINATED NODE   READINESS GATES
svclb-sample-server-load-balancer-8rxz5     1/1     Running             0          16m   10.42.0.132   localhost   <none>           <none>
sample-server-deployment-6449668b9c-rb4rg   0/1     ContainerCreating   0          3s    <none>        localhost   <none>           <none>
sample-server-deployment-6449668b9c-n77kz   0/1     ContainerCreating   0          3s    <none>        localhost   <none>           <none>
sample-server-deployment-6449668b9c-6sd9l   0/1     ContainerCreating   0          3s    <none>        localhost   <none>           <none>
```

Let's wait a bit and retry
```
$ kubectl --namespace sample-server-ns get pods -o wide
NAME                                        READY   STATUS    RESTARTS   AGE   IP            NODE        NOMINATED NODE   READINESS GATES
svclb-sample-server-load-balancer-8rxz5     1/1     Running   0          17m   10.42.0.132   localhost   <none>           <none>
sample-server-deployment-6449668b9c-6sd9l   1/1     Running   0          16s   10.42.0.135   localhost   <none>           <none>
sample-server-deployment-6449668b9c-n77kz   1/1     Running   0          16s   10.42.0.133   localhost   <none>           <none>
sample-server-deployment-6449668b9c-rb4rg   1/1     Running   0          16s   10.42.0.134   localhost   <none>           <none>
```

Great, they appear as running, now, let's curl this thing up (pun intended).

```
$ curl --dump-header - 192.168.122.76:31008
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.9.1
Date: Fri, 19 Feb 2021 15:10:06 GMT
Content-type: text/html; charset=utf-8
Content-Length: 982

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href="sample-server-deployment-6449668b9c-4wv7z-1613745329.txt">sample-server-deployment-6449668b9c-4wv7z-1613745329.txt</a></li>
<li><a href="sample-server-deployment-6449668b9c-6sd9l-1613747352.txt">sample-server-deployment-6449668b9c-6sd9l-1613747352.txt</a></li>
<li><a href="sample-server-deployment-6449668b9c-lq48c-1613745329.txt">sample-server-deployment-6449668b9c-lq48c-1613745329.txt</a></li>
<li><a href="sample-server-deployment-6449668b9c-n77kz-1613747351.txt">sample-server-deployment-6449668b9c-n77kz-1613747351.txt</a></li>
<li><a href="sample-server-deployment-6449668b9c-rb4rg-1613747353.txt">sample-server-deployment-6449668b9c-rb4rg-1613747353.txt</a></li>
</ul>
<hr>
</body>
</html>
```

✨✨✨

**AWESOME!!**
* ✅ We kept the previous two files from the old pods, and created 3 news from
the current replica. WE HAVE ACHIEVED PERSISTENT STORAGE!

✨✨✨

And if we run curl again several times the file names won't change:

```
$ curl --dump-header - 192.168.122.76:31008
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.9.1
Date: Fri, 19 Feb 2021 15:11:02 GMT
Content-type: text/html; charset=utf-8
Content-Length: 982

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href="sample-server-deployment-6449668b9c-4wv7z-1613745329.txt">sample-server-deployment-6449668b9c-4wv7z-1613745329.txt</a></li>
<li><a href="sample-server-deployment-6449668b9c-6sd9l-1613747352.txt">sample-server-deployment-6449668b9c-6sd9l-1613747352.txt</a></li>
<li><a href="sample-server-deployment-6449668b9c-lq48c-1613745329.txt">sample-server-deployment-6449668b9c-lq48c-1613745329.txt</a></li>
<li><a href="sample-server-deployment-6449668b9c-n77kz-1613747351.txt">sample-server-deployment-6449668b9c-n77kz-1613747351.txt</a></li>
<li><a href="sample-server-deployment-6449668b9c-rb4rg-1613747353.txt">sample-server-deployment-6449668b9c-rb4rg-1613747353.txt</a></li>
</ul>
<hr>
</body>
</html>

$ curl --dump-header - 192.168.122.76:31008
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.9.1
Date: Fri, 19 Feb 2021 15:11:03 GMT
Content-type: text/html; charset=utf-8
Content-Length: 982

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href="sample-server-deployment-6449668b9c-4wv7z-1613745329.txt">sample-server-deployment-6449668b9c-4wv7z-1613745329.txt</a></li>
<li><a href="sample-server-deployment-6449668b9c-6sd9l-1613747352.txt">sample-server-deployment-6449668b9c-6sd9l-1613747352.txt</a></li>
<li><a href="sample-server-deployment-6449668b9c-lq48c-1613745329.txt">sample-server-deployment-6449668b9c-lq48c-1613745329.txt</a></li>
<li><a href="sample-server-deployment-6449668b9c-n77kz-1613747351.txt">sample-server-deployment-6449668b9c-n77kz-1613747351.txt</a></li>
<li><a href="sample-server-deployment-6449668b9c-rb4rg-1613747353.txt">sample-server-deployment-6449668b9c-rb4rg-1613747353.txt</a></li>
</ul>
<hr>
</body>
</html>
```

✨✨✨

**AWESOME!!**

✨✨✨

Now a couple of notes:

**📝 NOTE:** The storage is "persistent" from the pods perspective, but it's
relaying on the cluster file storage, this can incurr in inavailablily. You can
change the storage class to something more "persistent", like a hard disk in AWS
(AWS EBS [Elastic Block Service] for example).

**📝 NOTE:** The diffs might be difficult to follow if you want to go step by
step :/ but is more about my train of thought/research than replicating exactly
my steps. Sorry 😅

**📝 NOTE:** had to change the port in our previous deployment `httpbin` so it
wouldn't clash with our current one. Now it lives in port `8100`.


<!-- Links for 1st chapter -->
[httpbin-namespace-definition-file]: ./httpbin/httpbin.yml
[httpbin-deployment-definition-file]: ./httpbin/httpbin_deployment.yml
[httpbin-service-definition-file]: ./httpbin/httpbin_service.yml
[httpbin-ingress-definition-file]: ./httpbin/httpbin_ingress.yml
[httpbin-traefik-k8s-annotations]: https://doc.traefik.io/traefik/v1.7/configuration/backends/kubernetes/#annotations
[config-best-practices-naked-post]: https://kubernetes.io/docs/concepts/configuration/overview/#naked-pods-vs-replicasets-deployments-and-jobs
<!-- Links for 2nd chapter -->
[sample_server-deployment-definition-file]: ./sample_server/sample_server_deployment.yml
[sample_server-service-definition-file]: ./sample_server/sample_server_service.yml
[sample_server-pv-claim-definition-file]: ./sample_server/sample_server_pv_claim.yml
