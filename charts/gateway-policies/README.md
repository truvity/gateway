# gateway-policies

The policies that protect what `gateway-fleet` and `gateway-groups` expose,
as Envoy Gateway objects rendered from generic values:

| Object | From | Default |
|---|---|---|
| `ClientTrafficPolicy` per namespace: the TLS floor and cipher set for every Gateway there, unless it carries the opt-out label | `tlsBaseline` | **on**, release namespace, TLS 1.2+, forward-secret AEAD ciphers |
| `ClientTrafficPolicy` on named Gateways or listeners | `tlsPolicies.<name>` | none |
| `SecurityPolicy` per protected route or listener: OIDC sign-in or JWT validation, the `authenticated` or `groups` posture, CSRF | `securityPolicies.<name>` + `defaults` | none; an entry without an issuer is refused |
| `BackendTLSPolicy`: verify a backend's private-chain certificate | `backendTLSPolicies.<name>` | none |

With an empty values file the chart renders one object, the TLS floor for
the release namespace, and authenticates nothing.

```sh
helm install policies oci://ghcr.io/truvity/charts/gateway-policies \
  --namespace gateways \
  --values policies-values.yaml
```

## Compatibility

Written for **Envoy Gateway v1.9.1** and the Gateway API it bundles: every
rendered test case was checked field by field against those CRD schemas.
Re-check on a controller upgrade. The SecurityPolicy `csrf` block, OIDC
`cookieConfig`, `endSessionEndpoint` and `ListenerSet` targets are recent
additions: older controllers reject or ignore them. `BackendTLSPolicy` is rendered at
`gateway.networking.k8s.io/v1` (Gateway API v1.4 and newer).

## Worked example

Neutral names throughout; an abridged copy of the `full` golden test case
(`tests/cases/gateway-policies/full/values.yaml`).

```yaml
commonAnnotations:
  example.com/sync-wave: "70"

tlsBaseline:
  namespaces: [gateways, example-tools]
  annotations:
    example.com/sync-wave: "66"

tlsPolicies:
  strict-listener:                  # one listener held to TLS 1.3 only
    namespace: gateways
    targetRefs:
      - name: public
        sectionName: partners
    tls: {minVersion: "1.3", maxVersion: "1.3"}

defaults:
  oidc:
    issuer: https://id.example.com
    authorizationEndpoint: https://id.example.com/authorize
    tokenEndpoint: https://id.example.com/token
    endSessionEndpoint: https://id.example.com/end_session
    scopes: [openid, email, profile, offline_access]
    idToken:
      enabled: true
      header: x-example-id-token
      providerName: example-id
      remoteJWKS:
        uri: https://id.example.com/keys
        backendRefs:                # keys from the issuer's Service next door
          - {name: issuer, namespace: example-id, port: 8080}
      claimToHeaders:
        - {header: x-auth-request-user, claim: sub}
        - {header: x-auth-request-email, claim: email}
  jwt:
    issuer: https://id.example.com
    remoteJWKS:
      uri: https://id.example.com/keys

securityPolicies:
  console:                          # anyone signed in; CSRF in shadow (default)
    namespace: example-console
    targetRefs: [{name: console}]
    oidc:
      clientID: example-console
      clientSecret: {name: example-console-client}
      hostname: console.example.com
      forwardAccessToken: true

  admin:                            # one group; CSRF enforced
    namespace: example-admin
    targetRefs: [{name: admin}]
    oidc:
      clientID: example-admin
      clientSecret: {name: example-admin-client}
      hostname: admin.example.com
      pathPrefix: /_auth
      cookieSameSite: Strict
    authorization:
      posture: groups
      groups: [example-admins]
    csrf:
      mode: enforce
      additionalOrigins: [https://portal.example.com]

  api:                              # a machine route: bearer tokens, no CSRF
    type: jwt
    namespace: example-api
    targetRefs: [{kind: GRPCRoute, name: api}]
    jwt:
      audiences: [example-api]
    authorization:
      posture: groups
      groups: [example-api-callers]

backendTLSPolicies:
  store:
    namespace: example-store
    targetRefs: [{name: store, sectionName: https}]
    hostname: store.example-store.svc
    caCertificateRefs: [{kind: ConfigMap, name: example-trust-bundle}]
```

Its render is `tests/golden/gateway-policies/full.yaml`.

## Values

Every key is checked by `values.schema.json`: a misspelt key is an error,
not a silent default. Relational mistakes (two policies on one route, a
`groups` posture with nothing to read groups from, CSRF on a machine route)
fail the render with a message naming the entry; each has a fixture under
`tests/invalid/gateway-policies/`.

### Common

| Value | Default | Notes |
|---|---|---|
| `commonAnnotations` | `{}` | on every object; an object's own `annotations` win per key |

### `tlsBaseline` — the floor for every Gateway

A ClientTrafficPolicy only attaches to Gateways in its own namespace, so
one is rendered per namespace. It selects Gateways by the **absence** of a
label, so a Gateway created later is covered from its first moment, and one
that needs something else opts out visibly by carrying the label.

| Value | Default | Notes |
|---|---|---|
| `tlsBaseline.enabled` | `true` | |
| `tlsBaseline.name` | `tls-baseline` | the same name in every namespace |
| `tlsBaseline.namespaces` | `[]` | empty: the release namespace only |
| `tlsBaseline.annotations`, `.labels` | `{}` | |
| `tlsBaseline.exemptLabel` | `tls-baseline-exempt` | a Gateway carrying this label key (any value) is not selected |
| `tlsBaseline.tls.minVersion` | `"1.2"` | `"1.0"`–`"1.3"` or `Auto`, quoted. RFC 8996 retired 1.0 and 1.1 |
| `tlsBaseline.tls.maxVersion` | `""` | empty renders nothing: Envoy's maximum (1.3) applies |
| `tlsBaseline.tls.ciphers` | the six ECDHE AEAD suites | TLS 1.2 and older only; not rendered when `minVersion` is `"1.3"` (the API refuses both). `[]` leaves the choice to Envoy. The default is Envoy's own list, stated so it is visible and does not move with a proxy upgrade |

**One Gateway, one Gateway-level policy.** A Gateway that already has its
own Gateway-level ClientTrafficPolicy (for example `gateway-fleet`'s
`exposures.<n>.clientTrafficPolicy`) must carry the opt-out label
(`gateway-fleet`: `exposures.<n>.labels`). Two policies at the same level
are a conflict the controller resolves by age, not by intent. A
listener-level policy (`tlsPolicies` with a `sectionName`) is more specific
and wins for its listener without an opt-out.

### `tlsPolicies.<name>` — stricter TLS where it is needed

| Value | Default | Notes |
|---|---|---|
| `enabled` | `true` | |
| `name` | the map key | |
| `namespace` | *required* | the Gateway's namespace |
| `annotations`, `labels` | `{}` | |
| `targetRefs` | *required* | `{kind, name, sectionName}`; `kind` `Gateway` (default) or `ListenerSet`; `sectionName` narrows it to one listener |
| `tls.minVersion` | `"1.3"` | |
| `tls.maxVersion` | `""` | |
| `tls.ciphers` | `[]` | as for the baseline |

### `defaults` and `securityPolicies.<name>` — protected routes

`defaults` is merged under every entry, key by key; an entry's own value
replaces it, including an empty one (`""`, `false`, `[]`). So an issuer, a
namespace or a CSRF posture shared by many routes is stated once.

Entry-only keys:

| Value | Default | Notes |
|---|---|---|
| `enabled` | `true` | |
| `type` | `oidc` | `oidc`: the gateway signs the browser in. `jwt`: every request carries a bearer token |
| `name` | the map key | |
| `targetRefs` | *required* | `{kind, name, sectionName}`; `kind` `HTTPRoute` (default), `GRPCRoute`, `Gateway` or `ListenerSet`. Two entries on one target are refused: the controller would apply the older and silently ignore the newer |
| `extraSpec` | `{}` | deep-merged over the generated spec, last, for fields this chart does not model |

Keys valid in both `defaults` and an entry:

| Value | Default | Notes |
|---|---|---|
| `namespace` | `""` | *required* (here or in the entry): the target's namespace |
| `annotations`, `labels` | `{}` | |

#### `oidc` — browser sign-in

The gateway runs the authorization code flow itself, refreshes the tokens
and keeps them in cookies it encrypts: no proxy, no session store, no
cookie key to manage.

| Value | Default | Notes |
|---|---|---|
| `oidc.issuer` | `""` | **required** on every oidc entry. Until it is set nothing signs in: an entry without one is refused, never rendered half-configured. `https://` only |
| `oidc.authorizationEndpoint`, `.tokenEndpoint` | `""` | written out instead of discovered, so the controller never has to reach the issuer to translate the policy; empty uses discovery |
| `oidc.endSessionEndpoint` | `""` | sign-out also ends the session at the issuer. Without it the cookie is cleared, the issuer session lives on, and the next page load signs the person straight back in |
| `oidc.clientID` | `""` | *required* |
| `oidc.clientSecret.name` | `""` | *required*: an Opaque Secret **in the policy's namespace** with the key `client-secret`. Delivering it is the caller's job; the chart only references it (with `group` and `kind` written out) |
| `oidc.hostname` | `""` | the redirect becomes `https://<hostname><pathPrefix>/callback`. One of `hostname` or `redirectURL` is *required*: it must equal the redirect registered at the issuer |
| `oidc.pathPrefix` | `/oauth2` | the callback is `<pathPrefix>/callback`, sign-out `<pathPrefix>/logout`. Pick one the application never serves |
| `oidc.redirectURL` | derived | set in full to override |
| `oidc.logoutPath` | `<pathPrefix>/logout` | |
| `oidc.scopes` | `[openid, email, profile]` | `openid` is always sent. Most issuers need `offline_access` for a refresh token |
| `oidc.refreshToken` | `true` | refresh underneath the session instead of signing the person out when the access token expires |
| `oidc.forwardAccessToken` | `false` | send the access token upstream as `Authorization: Bearer` |
| `oidc.cookieSameSite` | `Lax` | `Lax`, `Strict`, `None` or `""` (renders no `cookieConfig`). Stated, not left to the browser, whose default differs by browser and version. `Lax` still sends the cookie on a top-level link into the application; `Strict` suits one nothing links into |
| `oidc.idToken.enabled` | `false` | verify the ID token as a JWT on every request. *Required* for the `groups` posture and for `claimToHeaders` |
| `oidc.idToken.header` | `x-id-token` | the header the ID token is handed over in and read from |
| `oidc.idToken.providerName` | `oidc` | the JWT provider's name; the authorization rule refers to it |
| `oidc.idToken.audiences` | `[clientID]` | an ID token is minted for exactly one client |
| `oidc.idToken.remoteJWKS.uri` | `""` | *required* when enabled |
| `oidc.idToken.remoteJWKS.cacheDuration` | `300s` | |
| `oidc.idToken.remoteJWKS.backendRefs` | `[]` | `{name, namespace, port}`: fetch the keys from an in-cluster Service instead of the public URL (`uri` still names the path) |
| `oidc.idToken.claimToHeaders` | `[]` | `{header, claim}`: set from the verified token, overwriting whatever the request carried under that name |

#### `jwt` — machine routes

| Value | Default | Notes |
|---|---|---|
| `jwt.providerName` | `jwt` | |
| `jwt.issuer` | `""` | **required** on every jwt entry; until it is set nothing validates |
| `jwt.audiences` | `[]` | **required**: without one, every token the issuer ever signed, for anything, is accepted |
| `jwt.remoteJWKS.uri` | `""` | *required* |
| `jwt.remoteJWKS.cacheDuration` | `300s` | |
| `jwt.remoteJWKS.backendRefs` | `[]` | as for `oidc.idToken` — the in-cluster issuer's Service |
| `jwt.extractFrom` | `{}` | `{}` is Envoy Gateway's default, `Authorization: Bearer`. Otherwise `{headers: [{name, valuePrefix}], cookies, params}` |
| `jwt.claimToHeaders` | `[]` | |

#### `authorization` — who passes

| Value | Default | Notes |
|---|---|---|
| `authorization.posture` | `authenticated` | `authenticated`: anyone the issuer signed in or signed a token for. `groups`: `defaultAction: Deny` and one rule allowing a token whose `claim` holds one of `groups` |
| `authorization.claim` | `groups` | read from the verified token (the ID token on an oidc entry) as a string array |
| `authorization.groups` | `[]` | *required* for `groups`; an empty allow-list is a route nobody can use |
| `authorization.ruleName` | `allow-groups` | |

#### `csrf` — cross-site request forgery

The gateway compares the `Origin` (or, when absent, the `Referer`) of every
mutating request — `POST`, `PUT`, `PATCH`, `DELETE` — with the request's
own host. It is the second line after `SameSite`, not a replacement: it
catches a same-site attacker, an older browser and a cookie already sent.

| Value | Default | Notes |
|---|---|---|
| `csrf.mode` | `""` | `off`, `shadow`, `enforce`, or `""` for the type's default: **`shadow` on `oidc`**, `off` on `jwt`. On a `jwt` entry anything but `off` is refused: a machine client sends no `Origin`, so CSRF on an API or a webhook refuses every mutating request |
| `csrf.additionalOrigins` | `[]` | other origins allowed to post, e.g. `https://app.example.com` (Envoy compares host and port only). Refused with `off` |
| `csrf.shadowFraction` | `100/100` | shadow only: the share evaluated in shadow; the rest are enforced. Lower the numerator to roll enforcement out gradually |

**Shadow → enforce.** Shadow evaluates every mutating request, counts the
verdict and lets it through. Enforce answers a failing one with `403
Invalid origin`. Move a route to `enforce` once the counters and the logs
below show that everything it would refuse is something it should.

The counters are Envoy's CSRF filter statistics, one set per listener
(`http.<connection manager prefix>.csrf.*`; in Prometheus form
`envoy_http_csrf_*` with the prefix as the `envoy_http_conn_manager_prefix`
label). They count identically in shadow and in enforce:

| Counter | Counts |
|---|---|
| `request_valid` | mutating requests whose source origin matched |
| `request_invalid` | mutating requests that would be (shadow) or were (enforce) refused |
| `missing_source_origin` | the subset of those with neither `Origin` nor `Referer` — typically a script, a curl or a webhook, not a browser |

A counter says *how many*, per listener, and on a shared listener many
routes feed one counter. To say *which* request, add these fields to the
proxy's access log with `gateway-fleet` (v1.1.0 and newer,
`proxy.accessLog.extraFields`):

```yaml
accessLog:
  extraFields:
    origin: "%REQ(ORIGIN)%"
    referer: "%REQ_WITHOUT_QUERY(REFERER)%"   # never log the query: it can carry a code
    sec-fetch-site: "%REQ(SEC-FETCH-SITE)%"
    sec-fetch-mode: "%REQ(SEC-FETCH-MODE)%"
```

With the default fields (`method`, `:authority`, `route_name`,
`user-agent`, `response_code`, `response_code_details`) every shadow
decision can be recomputed from one line: a mutating `method` on a route
with the policy, whose `origin` — or, when empty, `referer` — does not
match `:authority` or an additional origin, is one the filter would refuse.
Empty on both is `missing_source_origin`. `sec-fetch-*` are set by browsers
and nothing else, so they separate a real cross-site form post from a
script. Once enforced, a refusal is logged with `response_code` 403 and the
filter's detail in `response_code_details`.

### `backendTLSPolicies.<name>` — private-chain backends

| Value | Default | Notes |
|---|---|---|
| `enabled` | `true` | |
| `name` | the map key | |
| `namespace` | *required* | the Service's namespace |
| `annotations`, `labels` | `{}` | |
| `targetRefs` | *required* | `{kind, name, sectionName}`; `kind` `Service` (default), `sectionName` the Service port's name |
| `hostname` | *required* | sent as SNI, and the name the certificate must carry |
| `caCertificateRefs` | `[]` | `{kind: ConfigMap\|Secret, name}` with the CA under `ca.crt`. Exactly one of this or `wellKnownCACertificates` |
| `wellKnownCACertificates` | `""` | `System`: the proxy's own trust store |
| `subjectAltNames` | `[]` | `{type: Hostname, hostname}` or `{type: URI, uri}`, verified instead of `hostname` |

The proxy presents a client certificate to such a backend only if its
EnvoyProxy has one (`gateway-fleet`: `proxy.backendTLS.clientCertificateRef`).
`gateway-groups` can also render a BackendTLSPolicy per group
(`groups.<n>.backendTLS`); use this chart for a backend that is not tied
to one group, and never both for one Service.

## What stays with the caller

The issuer, its clients and their secrets; the CA bundles; which routes
exist and which are protected. The chart renders policy from those inputs
and refuses combinations that cannot work; it never invents one.
