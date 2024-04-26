---
title: Dabbling with Envoy configurations - Part I
description: Exploring the different options that envoy provides and how it forms the basics of service meshes
slug: dabbling-with-envoy-configurations-part-I
date: 2021-06-28 00:00:00+0000
image: cover.jpg
categories:
    - servicemesh
tags:
    - envoy
    - reverseproxy
    - servicemesh
    - istio
---

## Types of envoy configurations

1. Static configuration
2. Dynamic configuration. This can be done in two ways:

    - From Filesystem
    - From Control plane

---

### Static configuration

To start envoy in static configuration we need the following:

1. [listeners](https://www.envoyproxy.io/docs/envoy/latest/start/quick-start/configuration-static#start-quick-start-static-listeners)
2. [clusters](https://www.envoyproxy.io/docs/envoy/latest/start/quick-start/configuration-static#start-quick-start-static-clusters)
3. [static_reources](https://www.envoyproxy.io/docs/envoy/latest/start/quick-start/configuration-static#start-quick-start-static-static-resources)
4. (Optional) [admin](https://www.envoyproxy.io/docs/envoy/latest/start/quick-start/admin#start-quick-start-admin) section

#### static_resources

Contain everything that is configured statically when envoy starts. Can contain the following:

- `[]listeners`
- `[]clusters`
- `[]secrets`

#### listeners

Lets configure an example listener on port 10000. Here all paths are matched and routed to `service_envoyproxy_io` cluster

```yaml
listeners:
- name: listener_0
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 10000
  filter_chains:
  - filters:
    - name: envoy.filters.network.http_connection_manager
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
        stat_prefix: ingress_http
        access_log:
        - name: envoy.access_loggers.stdout
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog
        http_filters:
        - name: envoy.filters.http.router
        route_config:
          name: local_route
          virtual_hosts:
          - name: local_service
            domains: ["*"]
            routes:
            - match:
                prefix: "/"
              route:
                host_rewrite_literal: www.envoyproxy.io
                cluster: service_envoyproxy_io
```

#### cluster

The service_envoyproxy_io cluster proxies over `TLS` to https://www.envoyproxy.io

```yaml
clusters:
- name: service_envoyproxy_io
  type: LOGICAL_DNS
  # Comment out the following line to test on v6 networks
  dns_lookup_family: V4_ONLY
  load_assignment:
    cluster_name: service_envoyproxy_io
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: www.envoyproxy.io
              port_value: 443
  transport_socket:
    name: envoy.transport_sockets.tls
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
      sni: www.envoyproxy.io
```

#### Testing this static configuration

If now we start envoy with this configuration using command `envoy -c <config_name>.yaml` and try querying the [localhost:10000](http://localhost:10000) port, we should get the envoyproxy homepage.

```shell
curl -v localhost:10000
```

---

### Dynamic Configuration from filesystem

In this setup Envoy will automatically update its configuration whenever the files are changed on the filesystem. The following sections are a must for dynamic configuration:

1. [node](https://www.envoyproxy.io/docs/envoy/latest/start/quick-start/configuration-dynamic-filesystem#start-quick-start-dynamic-fs-node)
2. [dynamic_resources](https://www.envoyproxy.io/docs/envoy/latest/start/quick-start/configuration-dynamic-filesystem#start-quick-start-dynamic-fs-dynamic-resources)

#### node

node needs a `cluster` and an `id`

```yaml
node:
  cluster: test-cluster
  id: test-id
```

#### dynamic_resources

Specifies where to load dynamic configuration from

```yaml
dynamic_resources:
  cds_config:
    path: ./cds.yaml
  lds_config:
    path: ./lds_yaml
```

#### `listener` resources

The linked `lds_config` should be an implementation of a [Listener Discovery Service](https://www.envoyproxy.io/docs/envoy/latest/configuration/listeners/lds#config-listeners-lds)

```yaml
resources:
- "@type": type.googleapis.com/envoy.config.listener.v3.Listener
  name: listener_0
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 10000
  filter_chains:
  - filters:
    - name: envoy.http_connection_manager
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
        stat_prefix: ingress_http
        http_filters:
        - name: envoy.router
        route_config:
          name: local_route
          virtual_hosts:
          - name: local_service
            domains:
            - "*"
            routes:
            - match:
                prefix: "/"
              route:
                host_rewrite_literal: www.envoyproxy.io
                cluster: example_proxy_cluster
```

#### `cluster` resources

The linked `cds_config` should be an implementation of a [Cluster Discovery Service](https://www.envoyproxy.io/docs/envoy/latest/configuration/upstream/cluster_manager/cds#config-cluster-manager-cds)

```yaml
resources:
- "@type": type.googleapis.com/envoy.config.cluster.v3.Cluster
  name: example_proxy_cluster
  type: STRICT_DNS
  connect_timeout: 3s
  typed_extension_protocol_options:
    envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
      "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
      explicit_http_config:
        http2_protocol_options: {}
  load_assignment:
    cluster_name: example_proxy_cluster
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: www.envoyproxy.io
              port_value: 443
  transport_socket:
    name: envoy.transport_sockets.tls
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
      sni: www.envoyproxy.io

```

#### Dynamically editing the configuration

Let's try editing this config to start proxying to google.com instead of envoyproxy.io  
In the `lds.yaml` file change the following:

```diff
            routes:
            - match:
                prefix: "/"
              route:
-               host_rewrite_literal: www.envoyproxy.io
+               host_rewrite_literal: www.google.com
                cluster: example_proxy_cluster
```

As soon as we do this write in the file, the LDS config in the envoy will update and will show in the logs:

```shell
lds: add/update listener 'listener_0'
```

We need to update the `cds.yaml` config as well:

```diff
  load_assignment:
    cluster_name: example_proxy_cluster
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
-              address: www.envoyproxy.io
+              address: www.google.com
              port_value: 443
  transport_socket:
    name: envoy.transport_sockets.tls
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
-      sni: www.envoyproxy.io
+      sni: www.google.com
```

We should see the similar update in envoy's logs about the CDS config update

```shell
cds: added/updated 1 cluster(s), skipped 0 unmodified cluster(s)
```

Hence we were able to reload the envoy configuration dynamically without restarting the server itself.
