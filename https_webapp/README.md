## Sample example of running webapp over https

In this example, we will serve the [webapp](https://github.com/Mohitsharma44/kubernetes-playground/tree/master/webapp)
over https.

We will also deploy a HAProxy as an ingress controller which will serve the incomming
https requests by performing ssl termination and based on the SNI, it will
forward the request to the `webapp-svc` (which is created as a part of webapp
deployment).
> For more information about the webapp, refer the repo link.

Some information for the demo:

**Webapp**

- The webapp container is listening on port 30000
- The webapp-svc (NodePort) service is also listening on port 30000
(and forwarding the requests to the backend webapps listening on port 30000)
- The webapp-svc service will serve 2 instances (`replicas`) of webapp

**HAproxy/ Ingress**

- The HAProxy image being used is pulled from `quay.io/jcmoraisjr/haproxy-ingress`
- This HAProxy needs a default backend service (for serving all requests that
doesn't match the forwarding rules), we will use `gcr.io/google_containers/defaultbackend:1.0`
- The Hostname will be `foo.bar`*
- The Ingress will be listening for HTTP and HTTPS (plus HAProxy's stat) requests

**RBAC**

- For Role Based Access Control, we will use the sample example from
`https://github.com/kubernetes/ingress/blob/master/examples/rbac/haproxy/ingress-controller-rbac.yml`

**TLS**

- We will use self signed certificate


> *Since we are using hostname as foo.bar and the HAProxy ingress will be using
> this SNI for forwarding the connetion to the service, you must add `foo.bar`
> in your /etc/hosts file.
> See the issue: https://github.com/kubernetes/ingress/issues/1303
> and this troubleshooting info: https://github.com/kubernetes/ingress/tree/master/examples/auth/client-certs/haproxy#troubleshooting


### Steps for running the demo:

#### Default Backend service
We run a default backend service to serve a 404 page

**Running default backend**
``` shell
kubectl run ingress-default-backend \
--image=gcr.io/google_containers/defaultbackend:1.0 \
--port=8080 \
--limits=cpu=10m,memory=20Mi \
--expose
```

**Making sure default backend service is running**

``` shell
kubectl get pods
```
and check for ingress-default-backend

#### Run Webapp

``` shell
kubectl create -f webapp.yml
```
This command will create the webapp deployment with
2 replicas and a webapp service.
> You can check the pods and services are running by
> running `kubectl get pods` and `kubectl get svc`

#### Preparing for Ingress

**Configmap**
Creating a `configmap` for HAProxy. For now, we'll just leave this empty
(HAProxy will use default values). To see wehat values can be changed, refer:
`https://github.com/jcmoraisjr/haproxy-ingress#configmap`
``` shell
kubectl create configmap haproxy-ingress
```

**RBAC**
``` shell
kubectl create -f ingress-controller-rbac.yml
```
The `ingress-controller-rbac` has all the permissions
for cluster and namespace to make sure the `ingress-controller`
can work

**Creating a TLS secret**

``` shell
openssl req \
-x509 -newkey rsa:2048 -nodes -days 365 \
-keyout tls.key -out tls.crt -subj '/CN=localhost'
```

**Creating a kubernetes secret**

``` shell
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key
```
We use secrets to share the tls certificate and key with the HAProxy Ingress

**Remove the keys** (once the secret is created, we dont need the original files)

``` shell
rm tls.crt tls.key
```

**Deploy HAproxy Ingress**

``` shell
kubectl create -f haproxy-ingress.yml
```
This will listen for HTTP on port 80, HTTPS on 443 and stat on 1936
(by default this is not exposed, we will expose this as a NodePort)

**Deploy HAProxy resource**

``` shell
kubectl create -f ingress-resource.yml
```
This will deploy the ingress resource which will forward the traffic based on rules
(in this case it is forwarding all requests for host `foo.bar` to `webapp-svc`
service listening on port 30000)
