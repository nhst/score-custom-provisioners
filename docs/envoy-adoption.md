# Adopting Envoy Gateway for an app

How an existing DN app moves from the nginx `Ingress` route to a Gateway API
`HTTPRoute` on the shared Envoy Gateway, using the `class: envoy` route
provisioner in this repo.

## What changes

| | Today | With Envoy |
|---|---|---|
| Score resource | `type: route` (no class) | `type: route`, `class: envoy` |
| Generated manifest | `networking.k8s.io/v1` `Ingress`, `ingressClassName: nginx` | `gateway.networking.k8s.io/v1` `HTTPRoute` |
| Attaches to | shared nginx ingress controller | `Gateway` `apps` in `envoy-gateway-system`, listener `https` |
| IP restriction | `ip-restriction` annotation on the Ingress | `ip-restriction: enabled` **label** + a namespace `SecurityPolicy` |
| Upstream vhost rewrite | `nginx.ingress.kubernetes.io/upstream-vhost` | none — Envoy forwards the original `Host` |

Nothing else in the app changes: same `Service`, same port, same container.

## Prerequisites

1. **Envoy Gateway must exist in the target cluster.** Verified present in
   `dev/blue` only (`dn-platform-argocd-config/clusters/dev/blue/gateway/`).
   Other clusters/colors have no `gateway/` folder — check with the platform
   team before writing an envoy route for `prod`.
2. **A `SecurityPolicy` must exist in your namespace** if you need IP
   restriction. Present in `dev/blue` for `subscription`, `editorial`, `infra`.
   Any other namespace: no policy → `ipRestriction: enabled` is a label
   nobody enforces → **the route is public**.
3. Your build already uses the reusable `score-k8s-template.yaml` workflow,
   which pulls this provisioner file:
   ```
   score-k8s init --provisioners \
     https://raw.githubusercontent.com/nhst/score-custom-provisioners/refs/heads/master/default-provisioner.yaml
   ```
   Nothing to install per app — the provisioner is fetched at build time.

## Step 1 — add the envoy route beside the nginx one

In `k8s/<app>/manifests/<env>/score-<env>.yaml`, keep `ingress` and add a
second resource. Both render while both exist, so this is a safe parallel run.

```yaml
resources:
  ingress:
    type: route
    params:
      host: my-app.apps.dev.dngroup.cloud
      port: 80
      path: /
      ipRestriction: enabled
  ingress-envoy:
    type: route
    class: envoy
    params:
      host: my-app.apps.dev.dngroup.cloud
      port: 80
      path: /
      ipRestriction: enabled
```

Params (same for both classes): `host`, `port`, `path`, `ipRestriction`,
`additional_hosts` (comma-separated), `env`.

`port` must be a **named service port number** from the workload's
`service.ports`, and `path` must start with `/` and not end with `/` —
the provisioner fails the build otherwise.

## Step 2 — verify the render

```
cd k8s/<app>
score-k8s init --provisioners https://raw.githubusercontent.com/nhst/score-custom-provisioners/refs/heads/master/default-provisioner.yaml \
  --file manifests/dev/score-dev.yaml
score-k8s generate manifests/dev/score-dev.yaml --namespace <your-namespace>
```

Expected HTTPRoute:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app
  namespace: editorial
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
    - my-app.apps.dev.dngroup.cloud
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: my-app
          port: 80
```

The HTTPRoute is named after the **service name**, the Ingress too — they are
different Kinds, so no name collision.

## Step 3 — deploy and check the route is accepted

Push, let CI commit to `cd/score-k8s-manifests`, let ArgoCD sync, then:

```
kubectl -n <namespace> get httproute my-app -o yaml
```

Look at `status.parents[].conditions` — `Accepted=True` and
`ResolvedRefs=True`. Common failures:

- `NotAllowedByListeners` — the `apps` Gateway's `https` listener does not
  allow routes from your namespace (platform-side `allowedRoutes` fix).
- `ResolvedRefs=False` — backend service/port name wrong.

## Step 4 — the DNS cutover (platform-team step, NOT automatic)

`*.apps.dev.dngroup.cloud` is a wildcard record that currently resolves to the
**nginx** load balancer. Checked 2026-09-15: a host with an HTTPRoute and a
host without one resolve to the same three IPs, and the response comes from
nginx. So:

> Creating an HTTPRoute does not move any traffic. Until DNS for the hostname
> points at the Envoy Gateway load balancer, every request still goes through
> nginx.

Two ways to actually exercise Envoy:

- **Test hostname**: give the envoy route a distinct host (e.g.
  `my-app-envoy.apps.dev.dngroup.cloud`) and have platform point that record
  at the Envoy Gateway LB. Real cutover risk stays zero.
- **Cutover**: platform repoints the app's record from the nginx LB to the
  Envoy Gateway LB. Rollback = repoint back, so keep the nginx `Ingress`
  resource in the score file until you are confident.

Get the Envoy LB address from the platform team
(`kubectl -n envoy-gateway-system get gateway apps` → `status.addresses`) —
that namespace is outside app-team access.

Also check whether anything upstream (Varnish / Fastly backend definitions)
names the app hostname; the backend hostname stays the same on a DNS cutover,
but TLS and client-IP handling at the new edge are worth a look before prod.

## Step 5 — remove nginx

Once traffic is served by Envoy and verified:

```yaml
resources:
  ingress:
    type: route
    class: envoy
    params: ...
```

Delete the old `ingress` resource (or rename the envoy one to `ingress`).
Next generate emits only the HTTPRoute; ArgoCD prunes the Ingress.

## IP restriction details

`ipRestriction: enabled` on an envoy route only adds the label
`ip-restriction: enabled`. Enforcement is the namespace `SecurityPolicy`
(`gateway.envoyproxy.io/v1alpha1`) which selects HTTPRoutes by that label and
applies `defaultAction: Deny` + an allow-list of client CIDRs
(`dn-platform-argocd-config/clusters/<env>/<color>/gateway/security-policy-<namespace>.yaml`).

Consequences:

- Any value of `ipRestriction` (not just `enabled`) adds the label — the
  provisioner only checks truthiness.
- No SecurityPolicy in your namespace → no enforcement → route is public.
  Ask platform to add one before moving a private app.
- Allow-list changes are a platform PR, not an app change.

## Known gaps

- Envoy does not receive the nginx `upstream-vhost` rewrite. Apps relying on
  the Host header being rewritten to `params.host` behave differently — but
  nginx only sets that annotation when `ipRestriction` or `env: test` is set,
  and it rewrites to the same `params.host`, so the practical difference is
  nil for normal hostnames. Verify if your app parses `Host`.
- One rule / one path prefix per route. Multi-path or header-based routing is
  not supported by the provisioner today; it needs a template change here.
- No TLS params — the route attaches to the Gateway's existing `https`
  listener and uses its certificate.
