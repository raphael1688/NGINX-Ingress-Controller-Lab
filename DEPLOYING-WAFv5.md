# Lab deployment

See the [prerequisites](/README.md#getting-started)

## Installing

1. Create NGINX Ingress Controller namespace

```code
kubectl create namespace nginx-ingress
```

2. Create Kubernetes secret to pull images from NGINX private registry

```code
kubectl create secret docker-registry regcred --docker-server=private-registry.nginx.com --docker-username=`cat <nginx-one-eval.jwt>` --docker-password=none -n nginx-ingress
```

Note: `<nginx-one-eval.jwt>` is the path and filename of your `nginx-one-eval.jwt` file

3. Create Kubernetes secret holding the NGINX Plus license

```code
kubectl create secret generic license-token --from-file=license.jwt=<nginx-one-eval.jwt> --type=nginx.com/license -n nginx-ingress
```

Note: `<nginx-one-eval.jwt>` is the path and filename of your `nginx-one-eval.jwt` file

4. List available NGINX Ingress Controller docker images that include NGINX App Protect WAF

```code
curl -s https://private-registry.nginx.com/v2/nginx-ic-nap/nginx-plus-ingress/tags/list --key <nginx-one-eval.key> --cert <nginx-one-eval.crt> | jq
```

Note: `<nginx-one-eval.key>` and `<nginx-one-eval.key>` are the path and filename of your `nginx-one-eval.crt` and `nginx-one-eval.crt` files respectively

Pick the latest version (`5.3.4` at the time of writing)

5. Apply NGINX Ingress Controller custom resources (make sure the URI below references the latest available `5.x` NGINX Ingress Controller version)

```code
kubectl apply -f https://raw.githubusercontent.com/nginx/kubernetes-ingress/v5.3.4/deploy/crds.yaml
kubectl apply -f https://raw.githubusercontent.com/nginx/kubernetes-ingress/v5.3.4/deploy/crds-nap-waf.yaml
```

6. Install NGINX Ingress Controller with NGINX App Protect through its Helm chart (set `nginx.image.tag` to the latest `5.x` available NGINX Ingress Controller version)

```code
helm install nic oci://ghcr.io/nginx/charts/nginx-ingress \
  --version 2.4.4 \
  -f deployment/values.yaml \
  -n nginx-ingress
```

7. Check NGINX Ingress Controller pod status

```code
kubectl get pods -n nginx-ingress
```

Pod should be in the `Running` state

```code
NAME                                            READY   STATUS    RESTARTS   AGE
nic-nginx-ingress-controller-7d758b8f8b-x4fvt   3/3     Running   0          39s
```

8. Check NGINX Ingress Controller logs

```code
kubectl logs -l app.kubernetes.io/instance=nic -n nginx-ingress -c nginx-ingress
```

Output should be similar to

```code
2026/02/17 11:14:36 [notice] 21#21: exit
2026/02/17 11:14:36 [notice] 20#20: exit
I20260217 11:14:36.367856   1 main.go:112] Event(v1.ObjectReference{Kind:"ConfigMap", Namespace:"nginx-ingress", Name:"nic-nginx-ingress", UID:"307a5ddf-dbcc-46a9-a2e0-c2b0886afdaa", APIVersion:"v1", ResourceVersion:"96841994", FieldPath:""}): type: 'Normal' reason: 'Updated' ConfigMap nginx-ingress/nic-nginx-ingress updated without error
I20260217 11:14:36.367890   1 main.go:112] Event(v1.ObjectReference{Kind:"ConfigMap", Namespace:"nginx-ingress", Name:"nic-nginx-ingress-mgmt", UID:"b2ea9af5-c927-45ed-9f2c-2012695bd747", APIVersion:"v1", ResourceVersion:"96841995", FieldPath:""}): type: 'Normal' reason: 'Updated' MGMT ConfigMap nginx-ingress/nic-nginx-ingress-mgmt updated without error
2026/02/17 11:14:36 [notice] 16#16: signal 17 (SIGCHLD) received from 21
2026/02/17 11:14:36 [notice] 16#16: worker process 21 exited with code 0
2026/02/17 11:14:36 [notice] 16#16: signal 29 (SIGIO) received
2026/02/17 11:14:36 [notice] 16#16: signal 17 (SIGCHLD) received from 20
2026/02/17 11:14:36 [notice] 16#16: worker process 20 exited with code 0
2026/02/17 11:14:36 [notice] 16#16: signal 29 (SIGIO) received
```

9. Check Kubernetes service status

```code
kubectl get svc -n nginx-ingress
```

NGINX Ingress Controller should be listening on TCP ports 80 and 443

```code
NAME                           TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
nic-nginx-ingress-controller   NodePort   10.103.68.87   <none>        80:32323/TCP,443:32254/TCP   63s
```

10. Check the `ingressclass`

```code
kubectl get ingressclass
```

The `nginx` ingressclass should be available

```code
NAME    CONTROLLER                     PARAMETERS   AGE
nginx   nginx.org/ingress-controller   <none>       74s
```

## Build the WAF compiler

1. List available F5 WAF for NGINX compiler versions
```code
curl -s https://private-registry.nginx.com/v2/nap/waf-compiler/tags/list --key <nginx-one-eval.key> --cert <nginx-one-eval.crt>
```

2. Set up Docker to authenticate to the privare registry
```code
mkdir -p /etc/docker/certs.d/private-registry.nginx.com
cp <nginx-one-eval.crt> /etc/docker/certs.d/private-registry.nginx.com/client.cert
cp <nginx-one-eval.key> /etc/docker/certs.d/private-registry.nginx.com/client.key
```

3. Log in to the Docker registry
```code
docker login private-registry.nginx.com
```

4. Build the F5 WAF for NGINX compiler:

```code
docker build --no-cache --platform linux/amd64 \
  --secret id=nginx-crt,src=<nginx-one-eval.crt> \
  --secret id=nginx-key,src=<nginx-one-eval.key> \
  -t waf-compiler-5.11.0:custom .
```

## Uninstalling

* Uninstall NGINX Ingress Controller through its Helm chart

```code
helm uninstall nic -n nginx-ingress
```

* Delete the namespace

```code
kubectl delete namespace nginx-ingress
```

* Delete custom resources

```code
kubectl delete -f https://raw.githubusercontent.com/nginx/kubernetes-ingress/v5.3.4/deploy/crds.yaml
kubectl delete -f https://raw.githubusercontent.com/nginx/kubernetes-ingress/v5.3.4/deploy/crds-nap-waf.yaml
```
