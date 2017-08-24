### Single Pod

Single nginx pod with port 80 exposed for HTTP traffic

We know that by default, the containers are not exposed to the cluster
and can only be accessed from the pod using `localhost`

So for exposing this container to the cluster there are two ways:

- Create a service (later we will look at what services are)
by running this command

``` shell
kubectl expose pod <podname> --type NodePort
```

Now you can describe the service

``` shell
# service_name in this case will be same as podname
kubectl describe svc <service_name>
```

find the IP of the service and run curl. You should see the nginx page


- Another way to access the pod would be to describe the pod

``` shell
kubectl describe pod <podname>
```
and get the hostname
or IP of the node where it is running, and curl the hostname or IP of that node
on the `port` that the service is binded to (you should get this when you
describe the service). You should see the nginx page
