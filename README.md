# Caddy Ingress Controller

This is the Kubernetes Ingress Controller for Caddy. It includes functionality for monitoring
Ingress resources on a Kubernetes cluster and includes support for providing automatic https
certificates for all hostnames defined in ingress resources that it is managing.

## Cloud Provider Setup (AWS, GCLOUD, ETC...)

In the Kubernetes folder a Helm Chart is provided to make installing the Caddy Ingress Controller
on a Kubernetes cluster straight forward. To install the Caddy Ingress Controller adhere to the
following steps:

1. Create a new namespace in your cluster to isolate all Caddy resources.

```sh
  kubectl apply -f ./kubernetes/deploy/00_namespace.yaml
```

2. Install the Helm Chart. (If you do not want automatic https set `autotls` to false and do not include
your email address as a value to the helm chart.)

```sh
  helm template \
    --namespace=caddy-system ./kubernetes/helm/caddyingresscontroller/ \
    --set autotls=true \
    --set email=youremail@test.com | kubectl apply -f -
```

The helm chart will create a service of type `LoadBalancer` in the `caddy-system` namespace on your cluster. You'll want to
set any DNS records for accessing this cluster to the external IP address of this LoadBalancer when the
external IP is provisioned by your cloud provider.

You can get the external IP address with `kubectl get svc -n caddy-system`

## Debugging

To view any logs generated by Caddy or the Ingress Controller you can view the pod logs of the Caddy Ingress Controller.

Get the pod name with:

```sh
  kubectl get pods -n caddy-system
```

View the pod logs:

```sh
kubectl logs <pod-name> -n caddy-system
```

## Automatic HTTPS

By default, any hosts defined in an ingress resource will configure caddy to automatically get certificates from let's encrypt and
will serve your side over HTTPS.

To disable automattic https you can set the argument `tls` on the caddy ingress controller to `false`.

Example:

Add args `tls=false` to the deployment.

```
  args:
    - -tls=false
```

## Bringing Your Own Certificates

If you would like to disable automatic HTTPS for a specific host and use your own certificates you can create a new TLS secret in Kubernetes and define
what certificates to use when serving your application on the ingress resource.

Example:

Create TLS secret `mycerts`, where `./tls.key` and `./tls.crt` are valid certificates for `test.com`.

```
kubectl create secret tls mycerts --key ./tls.key --cert ./tls.crt
```

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: example
  annotations:
    kubernetes.io/ingress.class: caddy
spec:
  rules:
  - host: test.com
    http:
      paths:
      - path: /
        backend:
          serviceName: test
          servicePort: 8080
  tls:
  - hosts:
    - test.com
    secretName: mycerts # use mycerts for host test.com
```