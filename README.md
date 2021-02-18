K8S Learning Ground!
===

This is gonna be my journal of learning k8s!

🚨 BEFORE YOU BEGIN 🚨
---

1. GET! SOME! K8s! (Minikube, k3s, anything you can run `kubectl` against will do.).


How to read this?
---

I'll help myself and whoever reads this not to do the classic "<s>Go to the YAML files and figure stuff out like the real hackers!</s>"

K8s is a VERY big beast, with a TON of moving parts. I'm not focused on learning how the cluster WORKS inside, I'd also like to learn German and my brain power is not infinite, so I'd like to get something more practical. I'll learn how to use it to deploy stuff (namely, how to structure/write YAML files that `kubectl` can use).

I'll do my best on keeping the files annotated so you (and future me) can follow through what's what.

⚠️ **WARNING** ⚠️
---

Whatever that you read CAN and PROBABLY WILL be inaccurate, I suck at reading docs, so mostly I just throw 9999 commands and then take notes on what worked. ALMOST EVERYTHING HERE is MY black-box presumption on what it is or happening.

_💬 yes, httpbin is my test application, too lazy to create one from scratch 😅_

**📝 IMPORTANT NOTE: for my brain sake, I've suffixed resource names, so they're much easily tracked through the source files.**

What should I do first? NAMESPACES!
---

1️⃣ let's go with Namespaces, that one is defined in [httpbin.yml][namespace-definition-file]. In K8s **namespaces isolate resources so they don't clash on each other**.

Lets imagine this Namespace as a tiny universe where our workload will be living.

**When you create a namespace, absolutely NOTHING** (besides the object being created) **happens**. Literally feels like you did NOTHING.

So **every resource** we'll use here, **WILL BE inside our** newly created **namespace**.

* 📝 **NOTE:** have a look at the metadata section, all K8S objects have that, it behaves like a `<head>` tag in a browser.
* 📝 **NOTE:** all K8S objects will need a `name`.
* 📝 **NOTE:** the labels section is not mandatory, but it can be used to filter resources, it doesn't hurt I guess 🤷.

To create the namespace, you can apply (💡 ahá, `apply` is a subcommand of `kubectl` that will create/update resources) the `httpbin.yml` file alone like:

```
$ kubectl apply -f httpbin.yml
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

Let's go the deployment route, reading naked in a documentation doesn't look correct, and the file definition doesn't seem to difficult either. You can see the deployment definition [here][deployment-definition-file].

* 📝 **NOTE:** see that the **namespace name** matches the one we created on [👉1️⃣](#what-should-i-read-first-namespaces)
* 📝 **NOTE:** `spec.selector.matchLabels` has to have labels that matches (the name implies it, duh) what's in `spec.template.metadata.labels`. This will be used by the ReplicaSet created under the hood.
  + 
* 📝 **NOTE:** `httpbin` exposes port 80, mind the `ports.containerPort` inside each container block.

Applying this file alone:

```
$ kubectl apply -f httpbin_deployment.yml 
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

Check out the service definition [here][service-definition-file].

* 📝 **NOTE:** see that the **namespace name** matches the one we created on [👉1️⃣](#what-should-i-read-first-namespaces)
* 📝 **NOTE:** the service behaves as an intermediary "container"/"pod" between the outside world and your pods. It'll link traffic from the service port (`spec.ports[0].port`) into the `pod` port (`spec.ports[0].targetPort`).
* 📝 **NOTE:** mind `spec.ports[0].targetPort` **has to match** `ports.containerPort` inside each container block.
* 📝 **NOTE:** `spec.selector.app` **has to match** whatever we wrote in the **[deployment definition file][deployment-definition-file] on key** `spec.template.metadata.labels`.

Let's create it now:

```
$ kubectl apply -f httpbin_service.yml
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

Let's see it [here in the ingress definition file][ingress-definition-file]

* 📝 **NOTE:** the full docs for [traefik annotations is here][traefik-k8s-annotations]
* 📝 **NOTE:** **SPECIAL ATTENTION** to `spec.rules[0].http.paths[0].backend.service.name` **gotta match** the load-balancer name!
* 📝 **NOTE:** **SPECIAL ATTENTION** to `spec.rules[0].http.paths[0].backend.service.port.number` **gotta match** the load-balancer exposed port! _(💡 AHÁ! here's used! 🤩)_

Let's apply this!

```
$ kubectl apply -f httpbin_ingress.yml 
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

[namespace-definition-file]: ./httpbin.yml
[deployment-definition-file]: ./httpbin_deployment.yml
[service-definition-file]: ./httpbin_service.yml
[ingress-definition-file]: ./httpbin_service.yml
[traefik-k8s-annotations]: https://doc.traefik.io/traefik/v1.7/configuration/backends/kubernetes/#annotations
[config-best-practices-naked-post]: https://kubernetes.io/docs/concepts/configuration/overview/#naked-pods-vs-replicasets-deployments-and-jobs
