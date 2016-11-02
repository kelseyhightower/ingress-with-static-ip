# Creating a Kubernetes Ingress Controller with a Static IP Address on GCP or GKE

This tutorial will walk you through creating a `nginx` deployment and expose it using a Kubernetes Ingress Controller associated with a static IP address on GKE. The use cases for doing this:

* You want to configure DNS before exposing your application to the outside world.
* Because you can.

This tutorial only works on GCP or GKE.

## TLDR;

Set the `kubernetes.io/ingress.global-static-ip-name` annontation on the Ingress config to the name of a Google Cloud Platform (GCP) global IP address as created with the `gcloud compute addresses create` command.

Example:

Create a global IP address:

```
gcloud compute addresses create kubernetes-ingress --global
```

Set the `kubernetes.io/ingress.global-static-ip-name` annotation on the Ingress config:

```
kind: Ingress
metadata:
  name: nginx
  annotations:
    kubernetes.io/ingress.global-static-ip-name: "kubernetes-ingress"
spec:
  backend:
    serviceName: nginx
    servicePort: 80
```

## Tutorial

Create a global IP address named `kubernetes-ingress`:

```
gcloud compute addresses create kubernetes-ingress --global
```

Create the `nginx` deployment:

```
kubectl run nginx --image nginx:1.11 --port 80 --replicas=3
```

Expose the `nginx` deployment using a NodePort service:

```
kubectl expose deployment nginx --type NodePort
```

> The service must be type NodePort to ensure the Ingress can perform health checks on the Pod.

Create the `nginx` ingress controller. Save the `nginx` ingress controller config to a file named `nginx-ing.yaml`:

```
cat > nginx-ing.yaml << EOF
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx
  annotations:
    kubernetes.io/ingress.global-static-ip-name: "kubernetes-ingress"
spec:
  backend:
    serviceName: nginx
    servicePort: 80
EOF
```

Create the `nginx` ingress controller:

```
kubectl create -f nginx-ing.yaml
```

After about 1 minute the `nginx` ingress controller should be ready for use and associated with the `kubernetes-ingress` global IP address created earlier. View the details of the `nginx` ingress controller:

```
kubectl get ing nginx
```
```
NAME      HOSTS     ADDRESS          PORTS     AGE
nginx     *         130.211.XX.XXX   80        59s
```

The `nginx` ingress address should match the `kubernetes-ingress` global IP address. Get the `kubernetes-ingress` IP address and compare it with the `nginx` ingress IP:

```
gcloud compute addresses \
  describe kubernetes-ingress --global \
  --format='value(address)'
```

```
130.211.XX.XXX
```

At this point you should be able to hit one of the `nginx` Pods via the `kubernetes-ingress` IP address. Store the `kubernetes-ingress` IP address in an environment variable:

```
export INGRESS_IP_ADDRESS=$(gcloud compute addresses \
  describe kubernetes-ingress --global \
  --format='value(address)')
```

Visit the `http://${INGRESS_IP_ADDRESS}` url:

```
curl http://${INGRESS_IP_ADDRESS}
```

HTTP Response:

```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
