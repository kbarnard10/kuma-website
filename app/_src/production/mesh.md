---
title: Create multiple service meshes in a cluster
content_type: how-to
---

This resource describes a very important concept in {{site.mesh_product_name}}, and that is the ability of creating multiple isolated service meshes within the same {{site.mesh_product_name}} cluster which in turn make {{site.mesh_product_name}} a very simple and easy project to operate in environments where more than one mesh is required based on security, segmentation or governance requirements.

Typically we would want to create a `Mesh` per line of business, per team, per application or per environment or for any other reason. Typically multiple meshes are being created so that a service mesh can be adopted by an organization with a gradual roll-out that doesn't require all the teams and their applications to coordinate with each other, or as an extra layer of security and segmentation for our services so that - for example - policies applied to one `Mesh` do not affect another `Mesh`.

`Mesh` is the parent resource of every other resource in {{site.mesh_product_name}}, including: 

* {% if_version lte:2.1.x %}[Data plane proxies](/docs/{{ page.version }}/explore/dpp){% endif_version %}{% if_version gte:2.2.x %}[Data plane proxies](/docs/{{ page.version }}/production/dp-config/dpp/){% endif_version %}
* [Policies](/policies)

In order to use {{site.mesh_product_name}} at least one `Mesh` must exist, and there is no limit to the number of Meshes that can be created. When a data plane proxy connects to the control plane (`kuma-cp`) it specifies to what `Mesh` resource it belongs: a data plane proxy can only belong to one `Mesh` at a time.

{% tip %}
When starting a new {{site.mesh_product_name}} cluster from scratch a `default` Mesh is being created automatically.
{% endtip %}

Besides the ability of being able to create virtual service mesh, a `Mesh` resource will also be used for:

* [Mutual TLS](/docs/{{ page.version }}/policies/mutual-tls/), to secure and encrypt our service traffic and assign an identity to the data plane proxies within the Mesh.
* [Traffic Metrics](/docs/{{ page.version }}/policies/traffic-metrics/), to setup metrics backend that will be used to collect and visualize metrics of our service mesh and service traffic within the Mesh.
* [Traffic Trace](/docs/{{ page.version }}/policies/traffic-trace/), to setup tracing backends that will be used to collect traces of our service traffic within the Mesh.
* {% if_version lte:2.1.x %}[Zone Egress](/docs/{{ page.version }}/explore/zoneegress){% endif_version %}{% if_version gte:2.2.x %}[Zone Egress](/docs/{{ page.version }}/production/cp-deployment/zoneegress/){% endif_version %}, to setup if `ZoneEgress` should be used for cross zone and external service communication.
* [Non-mesh traffic](/docs/{{ page.version }}/networking/non-mesh-traffic), to setup if `passthrough` mode should be used for the non-mesh traffic.

When [Mutual TLS](/docs/{{ page.version }}/policies/mutual-tls/) is enabled in `builtin` mode, each `Mesh` will provision its own CA root certificate and key unless we explicitly decide to use the same CA by sharing the same certificate and key across multiple meshes. When the CAs of our Meshes are different, data plane proxies from one `Mesh` will not be able to consume data plane proxies belonging to another `Mesh` and an intermediate API Gateway must be used in order to enable cross-mesh communication. {{site.mesh_product_name}} supports a [gateway mode](/docs/{{ page.version }}/explore/gateway) to make this happen.

## Usage

The easiest way to create a `Mesh` is to specify its `name`. The name of a Mesh must be unique.

{% tabs usage useUrlFragment=false %}
{% tab usage Kubernetes %}
```yaml
apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
```
We will apply the configuration with `kubectl apply -f [..]`.
{% endtab %}
{% tab usage Universal %}
```yaml
type: Mesh
name: default
```
We will apply the configuration with `kumactl apply -f [..]` or via the [HTTP API](/docs/{{ page.version }}/reference/http-api).
{% endtab %}
{% endtabs %}

### Creating resources in a Mesh

It is possible to determine to what `Mesh` other resources belong to in the following ways.

#### Data plane proxies

Every time we start a data plane proxy, we need to specify to what `Mesh` it belongs, this can be done in the following way:

{% tabs data-plane-proxies useUrlFragment=false %}
{% tab data-plane-proxies Kubernetes %}
By using the `kuma.io/mesh` annotation in a `Deployment`, like:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-app
  namespace: kuma-example
spec:
  ...
  template:
    metadata:
      ...
      annotations:
        # indicate to {{site.mesh_product_name}} what is the Mesh that the data plane proxy belongs to
        kuma.io/mesh: default
    spec:
      containers:
        ...
```

A `Mesh` may span multiple Kubernetes namespaces. Any {{site.mesh_product_name}} resource in the cluster which
specifies a particular `Mesh` will be part of that `Mesh`.

{% endtab %}
{% tab data-plane-proxies Universal %}

By using the `-m` or `--mesh` argument when running `kuma-dp`, for example:

```sh
kuma-dp run \
  --name=backend-1 \
  --mesh=default \
  --cp-address=https://127.0.0.1:5678 \
  --dataplane-token-file=/tmp/kuma-dp-backend-1-token
```
{% endtab %}
{% endtabs %}

You can control which data plane proxies are allowed to join the mesh using [mesh constraints]({% if_version gte:2.2.x %}/docs/{{ page.version }}/production/secure-deployment/dp-membership/{% endif_version %}{% if_version lte:2.1.x %}/docs/{{ page.version }}/security/dp-membership/{% endif_version %}).

#### Policies

When creating new [Policies](/policies) we also must specify to what `Mesh` they belong. This can be done in the following way:

{% tabs policies useUrlFragment=false %}
{% tab policies Kubernetes %}
By using the `mesh` property, like:

```yaml
apiVersion: kuma.io/v1alpha1
kind: TrafficRoute
mesh: default # indicate to {{site.mesh_product_name}} what is the Mesh that the resource belongs to
metadata:
  name: route-1
spec:
  ...
```

{{site.mesh_product_name}} consumes all [Policies](/policies) on the cluster and joins each to an individual `Mesh`, identified by this property.
{% endtab %}
{% tab policies Kubernetes (New policies) %}
By using the `kuma.io/mesh` label, like:

```yaml
apiVersion: kuma.io/v1alpha1
kind: MeshHTTPRoute
metadata:
  name: route-1
  namespace: {{site.mesh_namespace}}
  labels:
    kuma.io/mesh: default # indicate to {{site.mesh_product_name}} what is the Mesh that the resource belongs to
spec:
  ...
```

{{site.mesh_product_name}} consumes all [Policies](/policies) on the cluster and joins each to an individual `Mesh`, identified by this property.
{% endtab %}
{% tab policies Universal %}
By using the `mesh` property, like:
```yaml
type: TrafficRoute
name: route-1
mesh: default # indicate to {{site.mesh_product_name}} what is the Mesh that the resource belongs to
...
```
{% endtab %}
{% endtabs %}
