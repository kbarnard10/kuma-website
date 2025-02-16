---
title: Kubernetes Gateway API
---

{{site.mesh_product_name}} supports configuring [built-in gateway](/docs/{{ page.version }}/explore/gateway) using [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/).

## Installation

{% warning %}
[Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/) is still beta, therefore {{site.mesh_product_name}}'s integration provides the same level of stability.

Gateway API is not supported in multi-zone. To use the builtin Gateway, you need to use the [`MeshGateway` resources](/docs/{{ page.version }}/explore/gateway).
{% endwarning %}

1. Install the Gateway API CRDs.

   The Gateway API CRDs aren't available in Kubernetes by default yet. You must first [install the _experimental release_ that includes those CRDs as well as the admission webhook](https://gateway-api.sigs.k8s.io/guides/#install-experimental-channel).

2. Enable Gateway API support.

   - With `kumactl`, use the `--experimental-gatewayapi` flag.
   - With Helm, use the `experimental.gatewayAPI=true` value.

## Usage

1. Install the [counter demo](https://github.com/kumahq/kuma-counter-demo).

   ```sh
   kumactl install demo | kubectl apply -f -
   ```

2. Add a `Gateway`.

   The `Gateway` resource represents the proxy instance that handles traffic for a set of Gateway API routes.

   Every `Gateway` refers to a `GatewayClass` by name.

{% capture gateway %}
{% tabs usage useUrlFragment=false %}
{% tab usage Standard install %}

For Helm and `kumactl` installations, a `GatewayClass` named `kuma` is automatically installed
if the Gateway API CRDs are present.

{% endtab %}
{% tab usage Custom install %}

If you've installed {{site.mesh_product_name}} some other way, you can create your own `GatewayClass`
using the `controllerName: gateways.kuma.io/controller`:

```sh
echo "apiVersion: gateway.networking.k8s.io/v1beta1
kind: GatewayClass
metadata:
  name: kuma
spec:
  controllerName: gateways.kuma.io/controller
" | kubectl apply -f -
```

{% endtab %}
{% endtabs %}

```sh
echo "apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: kuma
  namespace: kuma-demo
spec:
  gatewayClassName: kuma
  listeners:
  - name: proxy
    port: 8080
    protocol: HTTP
" | kubectl apply -f -
```

When a user applies a `Gateway` resource, {{site.mesh_product_name}} automatically creates a `Deployment` of built-in gateways with a corresponding `Service`.

```
kubectl get pods -n kuma-demo
NAME                          READY   STATUS    RESTARTS   AGE
redis-59c9d56fc-6gcbc         2/2     Running   0          2m8s
demo-app-5845d6447b-v7npw     2/2     Running   0          2m8s
kuma-4j6wr-58998b5576-25wl6   1/1     Running   0          30s

kubectl get svc -n kuma-demo
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
redis        ClusterIP      10.43.223.223   <none>        6379/TCP         3m27s
demo-app     ClusterIP      10.43.216.203   <none>        5000/TCP         3m27s
kuma-pfh4s   LoadBalancer   10.43.122.93    172.20.0.3    8080:30627/TCP   87s
```
{% endcapture %}

{{ gateway | indent }}
The `Gateway` is now accessible using the external address `172.20.0.3:8080`.

3. Add an `HTTPRoute`.

   `HTTPRoute` resources contain a set of matching criteria for HTTP requests and upstream `Services` to route those requests to.

   ```sh
   echo "apiVersion: gateway.networking.k8s.io/v1beta1
   kind: HTTPRoute
   metadata:
     name: echo
     namespace: kuma-demo
   spec:
     parentRefs:
     - group: gateway.networking.k8s.io
       kind: Gateway
       name: kuma
       namespace: kuma-demo
     rules:
     - backendRefs:
       - group: ''
         kind: Service
         name: demo-app
         port: 5000
         weight: 1
       matches:
       - path:
           type: PathPrefix
           value: /
   " | kubectl apply -f -
   ```

   After creating an `HTTPRoute`, accessing `/` forwards a request to the demo app:

   ```sh
   curl 172.20.0.3:8080/ -i
   ```

   ```
   HTTP/1.1 200 OK
   x-powered-by: Express
   accept-ranges: bytes
   cache-control: public, max-age=0
   last-modified: Tue, 20 Oct 2020 17:16:41 GMT
   etag: W/"2b91-175470350a8"
   content-type: text/html; charset=UTF-8
   content-length: 11153
   date: Fri, 18 Mar 2022 11:33:29 GMT
   x-envoy-upstream-service-time: 2
   server: Kuma Gateway

   <html>
   <head>
   ...
   ```

## TLS termination

Gateway API supports TLS termination by using standard `kubernetes.io/tls` Secrets.

Here is an example

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-tls
  namespace: kuma-demo
type: kubernetes.io/tls
data:
  tls.crt: "MIIEOzCCAyO..." # redacted
  tls.key: "MIIEowIBAAKC..." # redacted
```

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: kuma
  namespace: kuma-demo
spec:
  gatewayClassName: kuma
  listeners:
    - name: proxy
      port: 8080
      hostname: test.kuma.io
      protocol: HTTPS
      tls:
        certificateRefs:
          - name: secret-tls
```

Under the hood, {{site.mesh_product_name}} CP copies the `Secret` to `{{site.mesh_namespace}}` namespace and converts it to {% if_version lte:2.1.x %}[{{site.mesh_product_name}} secret](/docs/{{ page.version }}/security/secrets){% endif_version %}{% if_version gte:2.2.x %}[{{site.mesh_product_name}} secret](/docs/{{ page.version }}/production/secure-deployment/secrets/){% endif_version %}.
It tracks all the changes to the secret and deletes it upon deletion of the original secret.

## Customization

Gateway API provides the `parametersRef` field on `GatewayClass.spec`
to provide additional, implementation-specific configuration to `Gateways`.
When using Gateway API with {{site.mesh_product_name}}, you can refer to a `MeshGatewayConfig` resource:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: GatewayClass
metadata:
  name: kuma
spec:
  controllerName: gateways.kuma.io/controller
  parametersRef:
    kind: MeshGatewayConfig
    group: kuma.io
    name: kuma
```

This resource has the same [structure as the `MeshGatewayInstance` resource](/docs/{{ page.version }}/explore/gateway#usage-1)
except that the `tags` field is optional.
With a `MeshGatewayConfig` you can then customize
the generated `Service` and `Deployment` resources.

## Multi-mesh

You can specify a `Mesh` for `Gateway` and `HTTPRoute` resources
by setting the [`kuma.io/mesh` annotation](/docs/{{ page.version }}/reference/kubernetes-annotations#kumaiomesh)
Note that `HTTPRoutes` must also have the annotation to reference a
`Gateway` from a non-default `Mesh`.

## Cross-mesh

[Cross-mesh gateways](/docs/{{ page.version }}/explore/gateway#cross-mesh) are supported with Gateway API.
You'll just need to create a corresponding `GatewayClass`
pointing to a `MeshGatewayConfig` that
sets `crossMesh: true`:

```yaml
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: GatewayClass
metadata:
  name: kuma-cross-mesh
spec:
  controllerName: gateways.kuma.io/controller
  parametersRef:
    group: kuma.io
    kind: MeshGatewayConfig
    name: default-cross-mesh
---
apiVersion: kuma.io/v1alpha1
kind: MeshGatewayConfig
metadata:
  name: default-cross-mesh
spec:
  crossMesh: true
```

and then reference it in your `Gateway`:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: kuma
  namespace: default
spec:
  gatewayClassName: kuma-cross-mesh
```

## Multi-zone

Gateway API isn't supported with multi-zone deployments, use {{site.mesh_product_name}}'s `MeshGateways`/`MeshGatewayRoutes` instead.

## How it works

{{site.mesh_product_name}} includes controllers that reconcile Gateway API CRDs and convert them into the corresponding {{site.mesh_product_name}} gateway CRDs.
This is why in the GUI, {{site.mesh_product_name}} `MeshGateways`/`MeshGatewayRoutes` are visible and not Kubernetes Gateway API resources.

Kubernetes Gateway API resources serve as the source of truth for {{site.mesh_product_name}} gateways and
any edits to {{site.mesh_product_name}} gateway resources are overwritten.
