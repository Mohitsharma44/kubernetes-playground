## WebApp

This is a simple webapp containing dashboard that was created for NYU CUSP's
Urban Observatory project. The actual standalone application can be found here:
https://github.com/Mohitsharma44/uodashboard

The application contains a python backend app that handles the authentication,
exposes API to upload the images (to be shown on dashboard) and a websocket API
for displaying images in realtime on the browser.

Our aim is to convert this application into a docker container that can be
controlled using kubernetes.

We have built an image from Dockerfile and pushed it to the dockerhub:
https://hub.docker.com/r/mohitsharma44/uodashboard/

In this example, we will look at writing a yml file for kubernetes deployment.
To accompany the deployment, we will also create a service that will expose
the container as a `NodePort` so that it can accessed from the cluster.

### Create deployment

Creating deployment is pretty straightforward. We just need to remember a few key things:
- python app uses port 8888 for HTTP / TCP traffic
- we need to scale the application horizontally

Based on our second requirement, it is pretty clear that we either need replica
set or Deployment. Deployment provides a nice layer of abstraction over rs whose
effects will be evident at later stage. For now, we'll just accept this with a
grain of salt and use deployment.

Our deployment file looks like this:

``` yaml
apiVersion: "apps/v1beta1"
kind: Deployment
metadata:
    name: webapp
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: webapp-demo
        env: test
    spec:
      containers:
        - name: uodashboard
          image: mohitsharma44/uodashboard-container:dev
          ports:
            - name: http
              containerPort: 30000
              protocol: TCP
```

The structure of the deployment is similar to replica sets.
- It begins with an `apiVersion`
declaration. Deployment is present in `apps/v1beta1` apiVersion
- Next, we pass some metadata information, in this case, name for our deployment
- Next, we specifiy the `spec` of the deployment which includes `replicas` for
horizontal replica of the pods. In the `template` section, we pass the metadata
for these pods which includes things like `labels` (which we will use to refer
these cluster of pods when creating services, among other things).
- Finally we specify the `spec` of the `containers` which includes the `name` and
the `image` (from dockerhub by default) that you will use and finally list of `ports`
that you need the containers to expose to the pods (not to the cluster). This
contains the optional `name`, `containerPort` which specifies the port that container
needs to expose to the pod and finally `protocol` which mentions the type of
protocol that will be used on this port.

### Create service

We know by now that a Pod is a collection of one or more containers and that the
containers share an IP address and port space and are allocated distinct IP address
that cannot be used for IPC. By default containers can only
expose their ports to the pods in which they are running.
(Multiple containers can communicate with each other via `localhost`).

To expose a container to outside the cluster / cluster-wide, we need to create
a `service`. A service is an abstraction that defines the set of pods selected by
the `labels` (the labels we created in metadata in deployment's yml file). When
individual pods in a deployment dies, and a new one is spawned (due to `replica`
in `deployment` and `rs`) the services can find them and include them. There are
different types of services like `NodePort` where the traffic coming on a
particular port is mapped to the containers listening on (probably different) port

A simple NodePort service looks like this:

``` yaml

apiVersion: v1
kind: Service
metadata:
  name: webapp-svc
  spec:
    type: NodePort
    selector:
      app: webapp-demo
      env: test
    ports:
      - name: http
        protocol: "TCP"
        port: 30000
        nodePort: 30000

```

In this case, we specify the `name` in `metadata` as the name of the service.
- In the spec, we specify the `type` of the service.
- In the `selector` section, we specify the labels that should be selected to
be bounded to this service (multiple services can bind same pods/ deployments)
- Finally we have `ports` where we mention the `protocol` that will be used for
communicating. We want the traffic comming on the NodePort` 30000 to be routed
to one of the containers listening on `port` (30000 in this case, where our
python app is listening for HTTP traffic)


### Run the setup

To run the deployment,

``` shell
kubectl create -f webapp.yml
```

To run the service on those deployments,

``` shell
kubectl create -f webapp-svc.yml
```

Finally, you can point your webbrowser to the node's IP address and port 30000
to access the webpage

### To teardown the setup

To teardown the deployment,

``` shell
kubectl delete deployment webapp
```

To teardown the service,

``` shell
kubectl delete svc webapp-svc
```

> svc is a shorthand for service
