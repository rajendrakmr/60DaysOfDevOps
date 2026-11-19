🚀 Kubernetes Ingress, TLS and Gateway API in one post

Ever wondered how outside traffic reaches the right Service inside your cluster? Here's a quick breakdown 👇

🔹 Ingress Controller
An Ingress is just a set of rules (host + path → Service). A controller is what actually enforces them: ingress-nginx, Traefik, HAProxy, Istio or a cloud ALB. Without a controller, an Ingress object does nothing.

🔹 Host and Path Routing
• Path-based: example.com/api → api-svc, example.com/ → web-svc
• Host-based: blog.example.com and shop.example.com on a single IP
• pathType: Exact | Prefix | ImplementationSpecific

🔹 TLS
Store the certificate in a kubernetes.io/tls Secret and reference it under spec.tls. With cert-manager + Let's Encrypt, certificates are issued and renewed automatically. One annotation is all it takes:
cert-manager.io/cluster-issuer: letsencrypt-prod

🔹 Gateway API
GatewayClass → Gateway → HTTPRoute
✅ Role separation: the platform team owns the Gateway, app teams own their routes
✅ Built-in traffic splitting (a 90/10 canary with no annotation hacks)
✅ Header matching, redirects, rewrites
✅ TCP and gRPC support
 
🛠 Debug tip:
kubectl describe ingress <name>
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller

📄 The full notes, with YAML examples, are attached as a PDF. Save it for later!

Are you still on Ingress, or have you already moved to Gateway API? Let me know in the comments 👇

#Kubernetes #DevOps #CloudNative #Ingress #GatewayAPI #TLS #CertManager #SRE #Learning