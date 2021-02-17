K8S Learning Ground!
===

This is gonna be my journal of learning k8s!

🚨 BEFORE YOU BEGIN 🚨
---

1. GET! SOME! K8s! (Minikube, k3s, anything you can run `kubectl` against will do.).


How to read this?
---

I'll help myself and whoever reads this not to do the classic "<s>Go to the yaml files and figure stuff out like the real hackers!</s>"

K8s is a VERY big beast, with a TON of moving parts. I'm not focused on learning how the cluster WORKS inside, I have limited amount of time in my life, so I'd like to get soemthing more practical. I'll learn how to use it to deploy stuff (namely, how to structure/write yaml files that `kubectl` can use).

I'll do my best on keeping the files annotated so you (and future me) can follow through what's what.

**WARNING**
---

Definitions, Research, comments, ALMOST EVERYTHING HERE is MY blackbox presumption on what it is. Concepts WILL be wrong, specially because I lack a lot of info.

What should I read first? NAMESPACES!
---

1️⃣ let's go with Namespaces, that one is defined in [httpbin.yml][namespace-definition-file]. In K8s namespaces isolate resources so they don't clash on each other.

When you create a namespace, absolutely NOTHING (besides the object being created) happens. Literally feels like you did NOTHING.

So every resource we'll use here, WILL BE inside our newly created namespace.

📝 NOTE: have a look at the metadata section, all K8S objects have that, it behaves like a `<head>` tag in a browser.

To create this file you can apply it isolated like:

```
$ kubectl apply -f httpbin.yml
```

_yes, httpbin is my test application, too lazy to create one from scratch 🤷_


`kubectl` will reply with something like:

```
namespace/httpbin-ns created
```

📝 NOTE: k3s had no issue, but rancher/longhorn replied with a message saying that the namespace was missing an annotation, but it also said it auto patched it? So I guess it solves itself.


As I said earlier, this has absolutely NO visible effect, other than we'll need `--namespace httpbin` to fetch resources. Like:

```
$ kubectl --namespace httpbin-ns get all
```

WHICH `kubectl` will say that we have nothing, well, because there's nothing!

Have something running, even if you can't access it yet
---

2️⃣ Second thing to look for. Having a container doing something!

In k8s jargon, a container as (like we know them from docker or lxc) is called a `pod`, and as said earlier it has A SHIT TON of complexity behind that I completely ignore :D. However, I do know a couple of things.

Although you can create a `pod` with kubectl like you were to do `docker run` it's discouraged in the documentation. [put a reference here]. They encourage you to have that pod configured with a set of settings that will allow you to scale up/down and married to services. For this we use a K8s Deployment.

A deployment will create underneath a ReplicaSet (which will be making sure we have those pods running no-matter-what) and Pods (well, what the replicaset controls).

You can see the deployment definition [here][deployment-definition-file]

* 📝 NOTE: see that the namespace name matches the one we created on 1️⃣
* 📝 NOTE: `spec.selector.matchLabels` has to have labels that matches (the name implies it, duh) what's in `spec.template.metadata.labels`. This will be used by the replicaset.
* 📝 NOTE: `httpbin` exposes port 80, mind the `ports.containerPort` inside each container block.

Applying this file alone:

```
$ kubectl apply -f httpbin_deployment.yml 
```

`kubectl` will reply saying:

```
deployment.apps/httpbin-deployment created
```

With this done, now we can fetch resources like we did before.

```
$ kubectl --namespace httpbin-ns get all
```
* 📝 NOTE: `--namespace` flag

and we'll have much more info now:
```
NAME                                      READY   STATUS    RESTARTS   AGE
pod/httpbin-deployment-6754d5d56f-5mfjn   1/1     Running   0          56s
pod/httpbin-deployment-6754d5d56f-222sn   1/1     Running   0          56s

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/httpbin-deployment   2/2     2            2           56s

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/httpbin-deployment-6754d5d56f   2         2         2       56s
```

let's try to reach one node from within the cluster. First, let's get the internal ip for it

```
$ kubectl --namespace httpbin-ns get pods -o wide
```
* 📝 NOTE: `--namespace` flag

which prints:

```
NAME                                  READY   STATUS    RESTARTS   AGE    IP           NODE        NOMINATED NODE   READINESS GATES
httpbin-deployment-6754d5d56f-5mfjn   1/1     Running   0          7m5s   10.42.0.80   localhost   <none>           <none>
httpbin-deployment-6754d5d56f-222sn   1/1     Running   0          7m5s   10.42.0.79   localhost   <none>           <none>
```

I'll take the first one `10.42.0.80` and spin a curl image so we throw an http request to it..

```
kubectl --namespace httpbin-ns run -i --tty --image curlimages/curl --rm curl sh
```
* 📝 NOTE: `--namespace` flag is **REALLY** important here
* 📝 NOTE: order matters, for whatever reason, `--rm` won't work if it's at first, putting it after `--image` worked 🤷

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

✅ GREAT! We have pods/containers! and they're working! ✅

Now we need some sort of way of exposing those containers to the outside world.

Connecting your deployment to the "outside world"
---

_Well, it's debatable if it's really the "ouside world" but it's not within the cluster, that's for sure._

3️⃣ to throw is a service. A service will allow our pods to be exposed. There are 2 main types for this:

* `NodePort`: as it's looks, it'll give you a port (in the cluster) that will be routed to the deployments.
* `LoadBalancer`: this one, according to the docs, exposes your deployment using a cloud provider load balancer. I guess this is more similar to an AWS ALB/ELB. I'll be using this one, although I don't quite sure get the difference between the two of them in my local k3s. Later I'll do tests in AWS.

Check out the service definition [here][service-definition-file].

* 📝 NOTE: see that the namespace name matches the one we created on 1️⃣
* 📝 NOTE: `spec.selector.app` has to match whatever we wrote in the [deployment definition file][deployment-definition-file] on key `spec.template.metadata.labels`
* 📝 NOTE: the service exposes port 8000, not quite sure where's that 8000, but seems to be internal.
* 📝 NOTE: mind `targetPort` has to match `ports.containerPort` inside each container block.

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
IP:                       10.43.5.203
IPs:                      10.43.5.203
LoadBalancer Ingress:     192.168.122.76
Port:                     http  8000/TCP
TargetPort:               80/TCP
NodePort:                 http  31565/TCP
Endpoints:                10.42.0.79:80,10.42.0.80:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

so according to this, if I throw a request to `LoadBalancer Ingress` I should get something:

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


Let's use that Port below `LoadBalancer Ingress`.

```
# in my machine
$ curl --dump-header - 192.168.122.76:8000/get
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

✅ GREAT NEWS ✅ our service is reachable from the outside of the cluster! ✅✅✅

Using request-rewriting! [optional]
---

So, k8s has a default http server running, that apparently I can exploit using an Ingress. An ingress tells the cluster to, well, **ingress** traffic to a service.

I'll base this heavily on what I have at hand which is k3s. in my k3s cluster `træfik` is used instead of `nginx` (as the official k8s docs). This will affect some metadata in the ingress definition. 

Let's see it [here in the ingress definition file][ingress-definition-file]


* 📝 NOTE: the full docs for [træfik annotations is here][traefik-k8s-annotations]
* 📝 NOTE: SPECIAL ATTENTION to `spec.rules[0].http.paths[0].backend.service.name` gotta match the load-balancer name!
* 📝 NOTE: SPECIAL ATTENTION to `spec.rules[0].http.paths[0].backend.service.port.number` gotta match the load-balancer exposed port! AHÁ! here's used! :D

let's apply this!

```
$ kubectl apply -f httpbin_ingress.yml 
```

Nothing much should happen, justa success message.

```
ingress.networking.k8s.io/httpbin-ingress created
```

let's inspect it:

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

✅ YES!! ✅ now we can reach the service without a port! :D

[namespace-definition-file]: ./httpbin.yml
[deployment-definition-file]: ./httpbin_deployment.yml
[service-definition-file]: ./httpbin_service.yml
[ingress-definition-file]: ./httpbin_service.yml
[traefik-k8s-annotations]: https://doc.traefik.io/traefik/v1.7/configuration/backends/kubernetes/#annotations