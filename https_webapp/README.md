## Sample example of running webapp over https

In this example, we will serve the [webapp](https://github.com/Mohitsharma44/kubernetes-playground/tree/master/webapp)
over https.

We will also deploy a HAProxy as an ingress controller which will serve the incomming
https requests by performing ssl termination and based on the SNI, it will
forward the request to the `webapp-svc` (which is created as a part of webapp
deployment).
> For more information about the webapp, refer the repo link.
