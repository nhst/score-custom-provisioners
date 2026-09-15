# score-custom-provisioners

Custom score-k8s provisioners for DN Media Group.

## Route provisioners

| URI | Match | Emits |
|---|---|---|
| `template://community-provisioners/ingress-route` | `type: route` (any class) | nginx `Ingress` |
| `template://dn-provisioners/envoy-route` | `type: route`, `class: envoy` | Gateway API `HTTPRoute` on `apps/envoy-gateway-system/https` |

Params for both: `host`, `port`, `path`, `ipRestriction`, `additional_hosts` (comma-separated).

## Sample: opt in to Envoy

Full migration guide: [docs/envoy-adoption.md](docs/envoy-adoption.md).

Keep the existing `ingress` resource and add a second route resource with `class: envoy`.
Both are emitted while both are defined. Remove `ingress` later to cut over.

```yaml
apiVersion: score.dev/v1b1
metadata:
  name: engagement-preferences-service
containers:
  app:
    image: .
service:
  ports:
    www:
      port: 8080
      targetPort: 8080
resources:
  ingress:
    type: route
    params:
      host: engagement-preferences-service.apps.dev.dngroup.cloud
      port: 8080
      path: /
      ipRestriction: enabled
  ingress-envoy:
    type: route
    class: envoy
    params:
      host: engagement-preferences-service.apps.dev.dngroup.cloud
      port: 8080
      path: /
      ipRestriction: enabled
```

Renders (`score-k8s generate score.yaml --namespace subscription`):

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: engagement-preferences-service
  namespace: subscription
  labels:
    ip-restriction: enabled
spec:
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: apps
      namespace: envoy-gateway-system
      sectionName: https
  hostnames:
    - engagement-preferences-service.apps.dev.dngroup.cloud
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: engagement-preferences-service
          port: 8080
```

`ipRestriction: enabled` only adds the label. Enforcement needs a `SecurityPolicy`
in the same namespace selecting `ip-restriction: enabled` HTTPRoutes
(see `dn-platform-argocd-config/clusters/<env>/<color>/gateway/`). No policy in the
namespace means the route is public.
