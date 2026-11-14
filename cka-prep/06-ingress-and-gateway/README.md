# 06 - Ingress and Gateway

Topics: Ingress controller, TLS, path/host routing, Gateway API overview

## Kyun chahiye?

Har app ke liye alag LoadBalancer Service mehnga aur unmanaged hai. Ingress/Gateway **L7 (HTTP/HTTPS)** routing deta hai: ek entry point, kai Services (host aur path ke basis par).

```
Client -> LB/NodePort -> Ingress Controller (Pod) -> Service -> Pods
```

## 1. Ingress

**Ingress object** sirf rules ka declaration hai. Kaam tabhi hota hai jab cluster me ek **Ingress Controller** chal raha ho (NGINX, Traefik, HAProxy, cloud-native controllers jaise AWS LB Controller, GKE Ingress). Controller by default install nahi hota.

> Note: community `ingress-nginx` project ke maintenance status ke baare me current announcements check kar lena (project retire hone ki baat chali thi). Naye projects me Gateway API ya actively maintained controller prefer karo.

### IngressClass
Kaun sa controller is Ingress ko handle kare:
```bash
k get ingressclass
```
Ingress me `spec.ingressClassName: nginx`. Default class annotation se set hoti hai.

### Basic Ingress (path routing)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app1-svc
            port:
              number: 80
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: app2-svc
            port:
              number: 80
```

### pathType
- `Exact`: bilkul same path
- `Prefix`: `/` se split element-wise prefix (`/app` matches `/app/x`, `/app` par)
- `ImplementationSpecific`: controller ke upar

### Host-based routing
```yaml
spec:
  ingressClassName: nginx
  rules:
  - host: shop.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: {service: {name: shop-svc, port: {number: 80}}}
  - host: blog.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: {service: {name: blog-svc, port: {number: 80}}}
```

`defaultBackend` unmatched traffic ke liye.

### Imperative
```bash
k create ingress web --class=nginx \
  --rule="shop.example.com/=shop-svc:80" \
  --rule="shop.example.com/api*=api-svc:8080"
```
(`*` suffix ka matlab Prefix.)

### TLS
```bash
k create secret tls web-tls --cert=tls.crt --key=tls.key
```
```yaml
spec:
  ingressClassName: nginx
  tls:
  - hosts: ["shop.example.com"]
    secretName: web-tls
  rules:
  - host: shop.example.com
    ...
```
Secret ingress ke **same namespace** me hona chahiye. **cert-manager** + Let's Encrypt se certificates auto-issue/renew ho sakte hain (annotation `cert-manager.io/cluster-issuer`).

Test:
```bash
curl -k https://shop.example.com --resolve shop.example.com:443:<node-ip>
curl -H "Host: shop.example.com" http://<node-ip>:<nodeport>/
```

## 2. Gateway API

Ingress ka **zyada expressive, role-oriented** successor (official Kubernetes SIG project, CRDs ke roop me install hote hain, core me built-in nahi). CKA ke naye curriculum me iska overview aata hai.

### Resources aur roles
| Resource | Kaun manage karta hai | Kaam |
|----------|----------------------|------|
| `GatewayClass` | Infra provider | Kaun sa controller/implementation |
| `Gateway` | Cluster operator | Listeners (port, protocol, TLS), kahan traffic aata hai |
| `HTTPRoute` (aur `GRPCRoute`, `TLSRoute`, `TCPRoute`) | App developer | Routing rules aur backend Services |

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: web-gw
spec:
  gatewayClassName: example-class
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
spec:
  parentRefs:
  - name: web-gw
  hostnames: ["shop.example.com"]
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-svc
      port: 8080
      weight: 90
    - name: api-svc-v2
      port: 8080
      weight: 10          # canary / traffic splitting
```

### Ingress vs Gateway API

| | Ingress | Gateway API |
|---|---------|-------------|
| Expressiveness | Basic, annotations pe nirbhar | Header match, traffic split, redirects built-in |
| Protocols | HTTP/HTTPS | HTTP, gRPC, TLS, TCP, UDP |
| Roles | Ek object | GatewayClass/Gateway/Route (infra vs app teams) |
| Portability | Annotations controller-specific | Standardized fields |
| Cross-namespace | Limited | `ReferenceGrant` |

```bash
k get gatewayclass,gateway,httproute -A
k describe gateway web-gw         # Status/conditions: Accepted, Programmed
```

Ingress se migrate karne ke liye `ingress2gateway` tool hai.

## Debug

```bash
k get ingress ; k describe ingress web-ingress        # Address aaya? backend endpoints?
k get pods -n ingress-nginx
k logs -n ingress-nginx deploy/ingress-nginx-controller
k get svc,ep app1-svc
```
- 404: host/path match nahi hua ya backend Service galat.
- 502/503: backend Pods ready nahi ya Service port galat.
- Address empty: controller Service ko external IP nahi mila (LoadBalancer pending).

## Common mistakes

- Ingress controller install nahi hai aur Ingress banaya.
- `ingressClassName` missing/galat.
- Service port number aur targetPort ka confusion.
- TLS Secret galat namespace me.
- `rewrite-target` annotation galat use (path prefix strip ke liye regex capture groups chahiye).

## Practice labs

1. Do apps deploy karo, ek Ingress se `/app1` aur `/app2` route karo.
2. Host-based routing banao aur `curl -H "Host: ..."` se test karo.
3. Self-signed cert banao (`openssl req -x509 ...`), TLS Secret me daalo aur Ingress me TLS lagao.
4. Gateway API CRDs install karke Gateway + HTTPRoute banao, 90/10 traffic split try karo.
5. Ek Ingress rule ko HTTPRoute me manually convert karo.
