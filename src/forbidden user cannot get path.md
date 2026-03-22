When either a user or a pod cannot authenticate with the API, we see an error of this form.  You can also see this when interacting manually using kubectl. This happens when your user authentication token is invalid for the operation you are trying to do.

"forbidden: User \"system:anonymous\" cannot get path \"/\"",


This means a request hit the API server without valid credentials (no token, no cert, nothing). The API server didn't reject it outright — it _authenticated_ it as `system:anonymous`, then the **authorization** layer (RBAC) said: "this identity has no permissions to do that."


![](Pasted%20image%2020260321102158.png)


When a request arrives at the API server, it walks through authenticators in this order:


![](Pasted%20image%2020260321123748.png)

Every request enters a chain of authenticator plugins. The API server tries each one in order. The **first** authenticator that returns success wins — the rest are skipped. If **none** succeed, the request becomes `system:anonymous`.


```go
authenticators := []authenticator.Request{}

// 1. Request header (proxy)
// 2. Client certificate (x509)
// 3. Bearer token (SA tokens, OIDC, webhook, bootstrap)
// 4. Anonymous (fallback)

return union.New(authenticators...), nil
```

Let's dive a bit into each of these authenticators.

Client certificates (x509)

The most fundamental method. When you run `kubeadm init`, it generates a CA and signs client certs for admin, controller-manager, scheduler. Your kubeconfig on a self-managed cluster almost certainly uses this.

The workflow happens in this order:

- TLS handshake first, before http.
- Client presents certificate during handshake 
- API server verifies cert against --client-ca-file 
- Extracts identity from cert fields: `- CN (Common Name) → username - O (Organization) → groups`

Bearer token (OIDC)

This is the standard way to authenticate users in production. An external identity provider (Google, Okta, Azure AD, Dex, Keycloak) issues JWT tokens. The API server validates them.

The flow usually goes like this.

- Human logs in to OIDC provider (browser flow)
- Provider issues a JWT id_token
- kubectl sends it: `Authorization: Bearer eyJhbGciOiJSUzI1NiIs...`

- API server validates:
   - Fetches provider's JWKS (public keys) from `{issuer-url}/.well-known/openid-configuration`
   - Verifies JWT signature
   - Checks: iss == --oidc-issuer-url
   - Checks: aud contains --oidc-client-id
   - Checks: exp > now (not expired)

- Extracts identity from claims:
```
    email claim → username ("senthil@company.com")
    groups claim → groups (["devops", "platform"])
```

Webhook token auth

The API server sends the bearer token to an external webhook service and asks "who is this?" An example is the aws-iam-authenticator.

Request header (authenticating proxy)

A reverse proxy sits in front of the API server, authenticates the user (SAML, LDAP, whatever), then passes the identity in HTTP headers.

- User hits proxy (e.g., OAuth2 Proxy, Dex, Keycloak Gatekeeper)
- Proxy authenticates user (SAML, LDAP, OAuth2)
- Proxy forwards to API server with headers:

```
   X-Remote-User: senthil@company.com
   X-Remote-Group: developers
   X-Remote-Group: platform-team
```

- API server verifies proxy's client cert against `--requestheader-client-ca-file` (prevents spoofing)
- Extracts username and groups from headers

Static token / static password

The oldest, simplest, and least secure methods. A CSV file on disk maps tokens or passwords to users. Deprecated and should never be used in production.

Finally, Fallback: anonymous authentication

If every authenticator in the chain returns "I don't recognize this," the request falls through to the anonymous handler.

The identity that comes _out_ of authentication looks the same regardless of method — it's always a `UserInfo` struct (from `staging/src/k8s.io/apiserver/pkg/authentication/user/user.go`):

For service account tokens, the flow is:

- Request arrives with header: `Authorization: Bearer <token>`
- The ServiceAccountTokenAuthenticator validates the JWT signature against the API server's public key
- It extracts the `sub` claim `system:serviceaccount:<namespace>:<name>`
- The user info becomes: username=`system:serviceaccount:production:my-app`, groups=`["system:serviceaccounts", "system:serviceaccounts:production", "system:authenticated"]`
- This identity goes to the RBAC authorizer

A Service Account by itself has **zero permissions**. It needs a RoleBinding or ClusterRoleBinding. Best understood using an example:

```yaml
# 0. The Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
---
# 1. The Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-reader
  namespace: monitoring
---
# 2. A Role (namespace-scoped permissions)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: read-pods
  namespace: monitoring
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
# 3. Bind the role to the service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: monitoring
subjects:
- kind: ServiceAccount
  name: pod-reader
  namespace: monitoring
roleRef:
  kind: Role
  name: read-pods
  apiGroup: rbac.authorization.k8s.io
---
# 4. A pod that uses this service account
apiVersion: v1
kind: Pod
metadata:
  name: monitoring-agent
  namespace: monitoring
spec:
  serviceAccountName: pod-reader
  containers:
  - name: agent
    image: curlimages/curl
    command: ["sh", "-c", "while true; do curl -s --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt -H \"Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)\" https://kubernetes.default.svc/api/v1/namespaces/monitoring/pods; sleep 30; done"]
```

Look at the command. Unlike the first command, it points to the cacert and bearer token, and accesses the pods in a particular namespace it has access to.


```
kubectl apply -f service-account-rbac.yaml
namespace/monitoring created
serviceaccount/pod-reader created
role.rbac.authorization.k8s.io/read-pods created
rolebinding.rbac.authorization.k8s.io/pod-reader-binding created
pod/monitoring-agent created
```

Now you can see what it can and cannot do.

```
kubectl auth can-i list pods --as=system:serviceaccount:monitoring:pod-reader -n monitoring

yes

kubectl auth can-i delete pods --as=system:serviceaccount:monitoring:pod-reader -n monitoring

no
```

The forbidden user error can also be seen in production.  It is when a pod's
process tries to call the Kubernetes API. It tries to read the token from
`/var/run/secrets/kubernetes.io/serviceaccount/token`. But something goes
wrong, the token goes missing, expired, or the API server rejects it. The
request falls through the entire authenticator chain and lands on
system:anonymous.


One of the common ways this can happen in production is when a
MutatingAdmissionWebhook can modify the pod spec after you submit it but before
it's persisted. 

It can change serviceAccountName or set automountServiceAccountToken: false.

You deploy:

```
  spec:
    serviceAccountName: my-app    # has RBAC bindings
```

Mutating webhook patches:

```
  spec:
    serviceAccountName: default   # has NO bindings
```

OR

```
    automountServiceAccountToken: false  # no token at all

```

In common use cases, this can happen when an Istio sidecar injector (mutates pod
spec, sometimes changes SA for sidecar permissions), OPA Gatekeeper or
Kyverno policies that enforce SA rules, custom admission webhooks that
"normalize" pods, or a Vault Agent injector (adds init containers, may change
SA to one with a Vault auth role) modify the pod spec.

These are commonly the case when a Kyverno or security controller webhook
mutated our pod's service account — the running pod lost access to the API
server.

How do we find this?

```
# See what ACTUALLY got persisted (not what you submitted)
kubectl get pod my-pod -o jsonpath='{.spec.serviceAccountName}'

# Compare with your deployment spec
kubectl get deploy my-app -o jsonpath='{.spec.template.spec.serviceAccountName}'

# If they differ → a mutating webhook changed it
```


![](Pasted%20image%2020260321132528.png)


A variant of this problem is when a webhook sets automountServiceAccountToken:
false. Your pod starts fine — the application runs. But the moment any code
tries to call the K8s API, there's no token file at all. The client library
falls back to no auth, and you get system:anonymous.
