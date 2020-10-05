---
authors: Andrew Lytvynov (andrew@gravitational.com)
state: draft
---

# RFD 5 - Kubernetes service

## What

Dedicated Kubernetes service in a Teleport process.

## Why

Kubernetes integration in Teleport has been implemented as an optional addon to
the Proxy service. This has several downsides:
- doesn't follow the "separation of concerns" principle; proxy bundles in:
  - public endpoint for Teleport users (both Web UI and CLI)
  - Kubernetes API gateway
  - reverse tunnel server for other services (like Nodes behind firewall/NAT or
    trusted clusters)
- pulls in all of the semantics of a Proxy service, like being a target for the
  discovery protocol
  - this makes exposing a Kubernetes API behind firewall much trickier

The only benefit from bundling the Kubernetes access with the Proxy is direct
access to the reverse tunnel server, for access to leaf Teleport clusters.

The solution is to add a separate Kubernetes Service to the `teleport` binary.
This fits with the existing model of Node service (SSH) and Application service
([AAP](https://github.com/gravitational/teleport/issues/3951)).
Proxy service becomes more of a "dumb" router in this model (not really "dumb",
there's still a lot of authn/z and audit complexity).

This RFD complements the [Kubernetes 5.0 enhancements
design](https://docs.google.com/document/d/1cS6J2d_xBcJMWPewWPjdOZyrDHLKdqmgF1QLLo4E1YI).

## Details

### Service definition

The new Kubernetes service will inherit all of the configuration and behavior
of the existing `kubernetes` section of `proxy_service`:

```yaml
# Old format:
proxy_service:
    kubernetes:
        enabled: yes
        public_addr: [k8s.example.com:3026]
        listen_addr: 0.0.0.0:3026
        kubeconfig_file: /secrets/kubeconfig
```

but will exist in a new top-level config:

```yaml
# New format:
kubernetes_service:
    enabled: yes # default "no"
    public_addr: k8s.example.com:3026
    listen_addr: 0.0.0.0:3026
    kubeconfig_file: /secrets/kubeconfig
```

In addition to keeping the existing fields, `kubernetes_service` adds several
new ones described in [Kubernetes 5.0 enhancements
design](https://docs.google.com/document/d/1cS6J2d_xBcJMWPewWPjdOZyrDHLKdqmgF1QLLo4E1YI):
- `k8s_cluster_name`
- `labels`
- `kubeconfig_file` - same field name, but all clusters from the config are
  parsed instead of just `current-context`.

The Kubernetes service implements all the common features:
- connecting to an Auth server directly or via a Proxy tunnel
- registration using join tokens
- heartbeats to announce presence
- audit logging

### Backwards compatibility

The new Kubernetes service will be preferred to the old Proxy sub-feature. But
the old format should still work. To do this, at startup the `teleport` binary
will translate the old config to the new service definition under the hood.
All existing configs should still work and behave as before.

To encourage users to migrate, all new config fields (`k8s_cluster_name` and
`labels`) will be added to the new service definition only.
`kubconfig_file` in the old section will behave as before - only extracting
`current-context` and registering that as `k8s_cluster_name`. In the new
section, all contexts will be extracted and registered.
