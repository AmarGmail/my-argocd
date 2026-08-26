# Argo CD on k3d — Exercise Summary

This exercise demonstrates installing Argo CD on a k3d Kubernetes cluster, connecting it to a GitHub repository, using GitOps synchronization policies, and exposing the Argo CD UI through Traefik Ingress.

---

## 1. Environment

Argo CD was installed on an EC2 instance running a k3d cluster.

```text
k3d cluster: hero-cluster
Servers:     1
Agents:      2
LoadBalancer: enabled
```

Argo CD was installed in the:

```text
argocd
```

namespace.

Verify:

```bash
kubectl get pods -n argocd
kubectl get deployment -n argocd
kubectl get svc -n argocd
```

All Argo CD components were successfully running.

---

## 2. Install Argo CD

Create the namespace:

```bash
kubectl create ns argocd
```

Install Argo CD:

```bash
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Verify:

```bash
kubectl get pods -n argocd
```

---

## 3. Install Argo CD CLI

The Argo CD CLI was installed on the EC2 instance.

Verify the client:

```bash
argocd version --client
```

Running:

```bash
argocd version
```

without specifying an Argo CD server results in:

```text
Argo CD server address unspecified
```

This is expected until the CLI is connected to an Argo CD server.

---

## 4. GitHub Repository

Argo CD was connected to:

```text
https://github.com/AmarGmail/my-argocd.git
```

Repository structure:

```text
my-argocd/
├── README.md
├── nginx_yaml_files/
│   ├── deployment.yaml
│   └── service.yaml
└── argocd-ingress/
    ├── argocd-ingressroute.yaml
    └── ...
```

The Kubernetes manifests under:

```text
nginx_yaml_files/
```

were used by Argo CD to deploy the Nginx application.

---

## 5. Argo CD Application

An Argo CD application named:

```text
nginx-app
```

was configured.

Important settings:

```text
Project:   default
Repository: https://github.com/AmarGmail/my-argocd.git
Path:       nginx_yaml_files
Cluster:    https://kubernetes.default.svc
Namespace:  default
```

Check the application:

```bash
argocd app get nginx-app
```

Important application states include:

```text
Synced
OutOfSync
Healthy
Missing
```

---

## 6. Kubernetes YAML Troubleshooting

Several YAML mistakes were encountered during the exercise.

### Incorrect labels

Incorrect:

```yaml
metadata:
  labels: nginx
```

Correct:

```yaml
metadata:
  labels:
    app: nginx
```

`metadata.labels` must be a key/value map.

### Incorrect containers field

Incorrect:

```yaml
container:
```

Correct:

```yaml
containers:
```

### Incorrect Service kind

Incorrect:

```yaml
king: Service
```

Correct:

```yaml
kind: Service
```

These errors demonstrated how Argo CD reports invalid Kubernetes manifests during synchronization.

---

# 7. Argo CD Sync Policies

The following Argo CD synchronization options were tested.

### Auto Sync

Automatically applies changes detected in Git.

```text
Git
 |
 v
Argo CD
 |
 v
Kubernetes
```

### Prune Resources

Removes Kubernetes resources that are no longer defined in Git.

For example:

```text
Resource exists in Kubernetes
Resource removed from Git
          |
          v
       Prune
          |
          v
Resource removed from Kubernetes
```

### Self Heal

Detects manual changes made directly to Kubernetes and restores the Git-defined state.

```text
Git desired state
       |
       v
    Argo CD
       |
       v
 Kubernetes

Manual change
       |
       v
Argo CD detects drift
       |
       v
State restored from Git
```

---

# 8. External Access — ClusterIP

Initially, the Argo CD server service was:

```text
argocd-server
TYPE: ClusterIP
```

ClusterIP provides internal Kubernetes access only.

---

# 9. External Access — NodePort

For experimentation, the service was changed to NodePort:

```bash
kubectl patch svc argocd-server -n argocd \
  -p '{"spec":{"type":"NodePort"}}'
```

Example result:

```text
80  -> 30329
443 -> 30599
```

Check:

```bash
kubectl get svc argocd-server -n argocd
```

This demonstrated Kubernetes NodePort-based external access.

---

# 10. External Access — Port Forward

Port forwarding was also tested:

```bash
kubectl port-forward --address 0.0.0.0 \
  svc/argocd-server -n argocd 8080:443
```

Port forwarding is useful for development and troubleshooting.

However, it is not the preferred approach for a persistent external deployment.

---

# 11. k3d LoadBalancer

The k3d cluster already had a server load balancer:

```text
k3d-hero-cluster-serverlb
```

The load balancer was configured to expose:

```text
EC2 :80
EC2 :443
```

Verify:

```bash
docker ps --format "table {{.Names}}\t{{.Ports}}"
```

The resulting architecture was:

```text
Internet
    |
    v
EC2 Public IP
    |
   :80 / :443
    |
    v
k3d LoadBalancer
    |
    v
Traefik
```

---

# 12. Traefik Ingress

k3d includes Traefik as the Kubernetes ingress controller.

Verify:

```bash
kubectl get pods -n kube-system | grep traefik
```

and:

```bash
kubectl get svc -n kube-system traefik
```

The goal was to route:

```text
https://EC2-PUBLIC-IP/
```

through Traefik to Argo CD.

---

# 13. TLS Problem

The first Traefik Ingress configuration produced:

```text
HTTP 500 Internal Server Error
```

Traefik logs showed:

```text
tls: failed to verify certificate:
x509: cannot validate certificate for 10.42.1.7
because it doesn't contain any IP SANs
```

The important discovery was that Argo CD itself was working correctly.

Testing from inside the cluster:

```bash
kubectl -n argocd run curl-test \
  --rm -it \
  --image=curlimages/curl \
  --restart=Never \
  -- \
  curl -vk https://argocd-server:443/
```

returned:

```text
HTTP/1.1 200 OK
```

Therefore the problem was specifically:

```text
Traefik
   |
   | HTTPS
   |
   X TLS certificate verification
   |
Argo CD
```

---

# 14. Traefik ServersTransport

A Traefik `ServersTransport` was created:

```yaml
apiVersion: traefik.io/v1alpha1
kind: ServersTransport

metadata:
  name: argocd-transport
  namespace: argocd

spec:
  insecureSkipVerify: true
```

This tells Traefik not to verify the backend TLS certificate.

> `insecureSkipVerify: true` is acceptable for this learning/lab environment. A production environment should use proper certificate validation.

---

# 15. Traefik IngressRoute

Instead of continuing with the standard Kubernetes `Ingress`, a Traefik-native `IngressRoute` was created.

Example:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute

metadata:
  name: argocd-server
  namespace: argocd

spec:
  entryPoints:
    - websecure

  routes:
    - match: PathPrefix(`/`)
      kind: Rule

      services:
        - name: argocd-server
          port: 443
          scheme: https
          serversTransport: argocd-transport
```

The important configuration is:

```yaml
scheme: https
serversTransport: argocd-transport
```

This explicitly tells Traefik to connect to Argo CD using HTTPS and the configured transport.

---

# 16. Successful HTTPS Test

After creating the `IngressRoute`:

```bash
kubectl apply -f argocd-ingress/argocd-ingressroute.yaml
```

verify:

```bash
kubectl get ingressroute -n argocd
```

Then:

```bash
curl -vk https://localhost/
```

returned:

```text
HTTP/2 200
content-type: text/html
<title>Argo CD</title>
```

This confirmed that Traefik was successfully routing HTTPS traffic to Argo CD.

---

# 17. Final Architecture

The final working architecture is:

```text
                         Internet
                            |
                            v
                    EC2 Public IP
                            |
                        HTTPS :443
                            |
                            v
                   k3d serverlb
                            |
                            v
                         Traefik
                            |
                       IngressRoute
                            |
                    ServersTransport
                 insecureSkipVerify=true
                            |
                            v
                  argocd-server :443
                            |
                            v
                         Argo CD
```

---

# 18. GitOps Application Flow

The complete GitOps workflow is:

```text
                   GitHub
                     |
                     | Kubernetes manifests
                     v
                  Argo CD
                     |
                     | Sync
                     v
               Kubernetes/k3d
                     |
                     v
              Nginx Deployment
                     |
                     v
                Nginx Pods
```

Argo CD continuously compares:

```text
Git desired state
        vs
Kubernetes actual state
```

and reports whether the application is:

```text
Synced
OutOfSync
Healthy
Missing
```

---

# 19. Access Methods Compared

During the exercise, several methods of accessing Argo CD were explored.

| Method | Purpose |
|---|---|
| ClusterIP | Internal Kubernetes access |
| Port Forward | Temporary development/debugging |
| NodePort | Simple external access |
| Traefik Ingress | Kubernetes ingress-based access |
| Traefik IngressRoute | Traefik-native ingress configuration |

The final solution uses:

```text
EC2
  |
  v
k3d LoadBalancer
  |
  v
Traefik
  |
  v
IngressRoute
  |
  v
Argo CD
```

---

# 20. Key Concepts Learned

This exercise covered:

- Argo CD installation
- Argo CD CLI
- GitOps
- Argo CD Applications
- Git repository integration
- Manual Sync
- Auto Sync
- Prune
- Self Heal
- Kubernetes YAML troubleshooting
- ClusterIP
- NodePort
- Port Forward
- k3d LoadBalancer
- Traefik
- Kubernetes Ingress
- Traefik IngressRoute
- Backend HTTPS
- TLS certificate verification
- Traefik ServersTransport
- Kubernetes desired state vs actual state

---

# 21. Next Steps

Possible improvements for the next iteration:

1. Change `argocd-server` back from NodePort to ClusterIP.

```bash
kubectl patch svc argocd-server -n argocd \
  -p '{"spec":{"type":"ClusterIP"}}'
```

2. Configure a hostname, for example:

```text
argocd.example.com
```

3. Configure a proper TLS certificate.

4. Replace the Traefik default self-signed certificate.

5. Configure Argo CD CLI login through the Ingress endpoint.

6. Deploy the Nginx application completely through GitOps.

7. Enable:

```text
Auto Sync
Prune
Self Heal
```

---

# 22. Final Result

The exercise successfully established a basic end-to-end GitOps environment:

```text
                         GitHub
                            |
                            | manifests
                            v
                         Argo CD
                            |
                            | Sync
                            v
                       Kubernetes
                          /   \
                         /     \
                        v       v
                   Deployment  Service
                        |
                        v
                      Nginx
```

External Argo CD access:

```text
Internet
    |
    v
EC2 Public IP
    |
    v
k3d LoadBalancer
    |
    v
Traefik
    |
    v
IngressRoute
    |
    v
Argo CD
```

Argo CD is now accessible through Traefik Ingress rather than relying on port forwarding or NodePort.
