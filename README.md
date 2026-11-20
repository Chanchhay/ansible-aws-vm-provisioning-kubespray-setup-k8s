# Self-Managed Kubernetes on AWS EC2 — HTTPS Application Hosting

A hands-on DevOps lab that provisions a multi-node Kubernetes cluster on AWS EC2 using **Ansible and Kubespray**, exposes applications through a **self-hosted NGINX TCP load balancer and F5 NGINX Ingress Controller**, and provides browser-trusted HTTPS using **cert-manager, Let's Encrypt, and Cloudflare DNS**.

> **Lab status (20 November 2026):** Both applications returned `HTTP 200` over HTTPS with a publicly trusted Let's Encrypt certificate. HTTP-to-HTTPS redirection on port 80 is **not yet verified**; the latest external HTTP tests hung and require additional diagnosis.
>
> This is a **production-like learning environment**, not a production-grade highly available deployment. In particular, the public entry point depends on a single worker node.

## 1. Objectives

- Provision and operate Kubernetes on self-managed Ubuntu EC2 instances.
- Use Ansible and Kubespray rather than a managed Kubernetes service.
- Learn Kubernetes Deployments, Services, NodePorts, and Ingress resources.
- Route multiple domains to different services through a single public endpoint.
- Understand TCP load balancing versus TLS termination and HTTP routing.
- Automate trusted TLS certificate issuance and renewal with cert-manager.
- Integrate public DNS and ACME DNS-01 validation with Cloudflare.
- Diagnose networking, certificates, service discovery, and ingress failures.

## 2. Technology Stack

| Layer | Technology | Purpose |
| --- | --- | --- |
| Infrastructure | AWS EC2, VPC, Security Groups | Virtual machines and network access |
| Operating system | Ubuntu 24.04 | Kubernetes node hosts |
| Provisioning | Ansible, Kubespray | Configure and deploy the cluster |
| Container orchestration | Kubernetes | Schedule and manage applications |
| Package management | Helm | Install cluster components |
| Public load balancing | Self-hosted NGINX (`stream` / TCP proxy) | Forward ports 80 and 443 to Kubernetes NodePorts |
| Ingress | F5 NGINX Ingress Controller | TLS termination and hostname-based routing |
| Workloads | React, NGINX welcome page | Demonstration applications |
| DNS | Cloudflare | Public DNS records for application subdomains |
| Certificate authority | Let's Encrypt | Publicly trusted TLS certificates |
| Certificate automation | cert-manager | Obtain, store, and renew TLS certificates |

## 3. Infrastructure

**AWS region:** `ap-southeast-2` (Sydney)
**Cluster size:** 4 EC2 instances / 2 control-plane nodes / 2 worker nodes
**VPC private CIDR:** `172.31.0.0/16`

| Node | Private IP | Kubernetes role | Additional responsibility |
| --- | --- | --- | --- |
| `master01` | `172.31.8.195` | Control plane | etcd member |
| `master02` | `172.31.44.25` | Control plane | etcd member |
| `worker01` | `172.31.1.203` | Worker | etcd member; NodePort upstream |
| `worker02` | `172.31.44.17` | Worker | Public-facing NGINX edge proxy; NodePort upstream |

The three etcd members are `master01`, `master02`, and `worker01`. Placing etcd on a worker is a lab-specific compromise, not the default recommendation for a production environment.

### Important: public IP addresses change

This lab uses EC2 public IPv4 addresses without a permanently allocated public endpoint. **The public IP can change when an instance is stopped and started.** A previously observed public IP for `worker02` was `16.179.161.151`, but **do not assume it is still correct**.

After restarting instances:

1. Obtain each node's current EC2 public IP.
2. Refresh the Ansible inventory and any local Kubernetes API endpoint configuration that depends on a public IP.
3. Update the two Cloudflare A records to point at the *current* `worker02` public IP.
4. Verify SSH, `kubectl`, and public DNS resolution before troubleshooting application routing.

An Ansible inventory refresh playbook (`refresh-inventory.yaml`) was used during the lab; adjust its location and EC2 profile to match your repository.

## 4. How Traffic Flows

### HTTPS request path

```text
User / Browser
    |
    |  https://k8s.kagea.site  or  https://k8s-nginx.kagea.site
    v
Cloudflare DNS (DNS-only A records)
    |
    |  Both hostnames resolve to the worker02 public IPv4 address
    v
worker02: Self-hosted NGINX edge load balancer (host-networked Pod)
    |
    |  TCP/443 passthrough (TLS remains encrypted)
    v
Kubernetes NodePort 30443 on worker01 / worker02
    |
    v
F5 NGINX Ingress Controller
    |  - Terminates TLS using Kubernetes Secret: react-demo/kagea-site-tls
    |  - Matches the HTTP Host header against Ingress rules
    |
    +---- k8s.kagea.site ------> react-service:80 ------> React Pods
    |
    +---- k8s-nginx.kagea.site -> nginx-service:80 ------> NGINX Pods
```

The HTTP path is intended to be similar: edge NGINX port `80` forwards to NodePort `30080`, where Ingress can redirect HTTP to HTTPS. **The public port 80 path was not confirmed working in the latest tests.**

### Separate certificate-management path

```text
cert-manager -> Let's Encrypt ACME API
      |
      +-> Cloudflare API: create DNS-01 TXT challenge record
      |
      +-> Let's Encrypt verifies DNS ownership of kagea.site
      |
      +-> cert-manager writes the issued certificate and key
                 to Secret react-demo/kagea-site-tls
```

**Key distinction:** Cloudflare is used as the authoritative DNS provider and DNS-01 challenge provider. With **DNS only** enabled, application HTTPS connections go directly to the EC2 public endpoint; Cloudflare is not terminating their TLS.

*A standalone illustrated architecture diagram will be added separately.*

## 5. DNS and Application Routing

Cloudflare is the DNS provider for `kagea.site` (the domain is registered through Namecheap).

| DNS type | Name | Target | Cloudflare proxy |
| --- | --- | --- | --- |
| `A` | `k8s` | Current `worker02` public IP | DNS only (gray cloud) |
| `A` | `k8s-nginx` | Current `worker02` public IP | DNS only (gray cloud) |

**Public application URLs:**

- React: <https://k8s.kagea.site>
- NGINX welcome: <https://k8s-nginx.kagea.site>

Both hostnames deliberately resolve to the **same IP**. The Ingress Controller routes them to different Kubernetes Services based on the request hostname.

### Ingress configuration

Example `k8s/ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: react-ingress
  namespace: react-demo
  annotations:
    nginx.org/ssl-redirect: "false" # Set to "true" after HTTP port 80 is fixed
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - k8s.kagea.site
        - k8s-nginx.kagea.site
      secretName: kagea-site-tls
  rules:
    - host: k8s.kagea.site
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: react-service
                port:
                  number: 80
    - host: k8s-nginx.kagea.site
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-service
                port:
                  number: 80
```

This example uses the currently verified TLS Secret. The redirect annotation is intentionally left `false` in the documented baseline because HTTP port 80 still needs verification. If the cluster already uses `"true"`, it is fine to keep it after confirming port 80 works.

The two backing Services are **ClusterIP** services in the `react-demo` namespace. The public client does not connect directly to those ClusterIP addresses.

## 6. Kubernetes Components

| Namespace | Component | Role |
| --- | --- | --- |
| `react-demo` | `react-app` | React Deployment (observed: 2 replicas) |
| `react-demo` | `react-service` | Internal React ClusterIP Service, port 80 |
| `react-demo` | `nginx-welcome` | NGINX welcome Deployment |
| `react-demo` | `nginx-service` | Internal NGINX ClusterIP Service, port 80 |
| `react-demo` | `react-ingress` | Host-based routing and TLS reference |
| `react-demo` | `kagea-site-certificate` | cert-manager Certificate resource |
| `react-demo` | `kagea-site-tls` | Issued certificate/key Secret |
| `nginx-system` | `nginx-controller` | F5 NGINX Ingress Helm release |
| `nginx-system` | `nginx-controller-nginx-ingress-controller` | Ingress Controller NodePort Service |
| `edge-system` | `edge-nginx` | External TCP proxy workload on `worker02` |
| `cert-manager` | cert-manager components | Certificate issuance, renewal, admission webhook |
| Cluster scope | `letsencrypt-prod` | ACME ClusterIssuer |

### Relevant ingress Service ports

```text
HTTP:   Service port 80  -> NodePort 30080
HTTPS:  Service port 443 -> NodePort 30443
```

The edge NGINX TCP proxy forwards requests to the private worker IPs on these NodePorts. Since it forwards encrypted TCP bytes on port 443, it **does not own the application certificate**; the Ingress Controller does.

### Do not overwrite Kubespray's node-local NGINX config

Kubespray also runs a separate `nginx-proxy` **static Pod** on worker nodes to proxy access to the Kubernetes API server. Its generated configuration may be located at `/etc/nginx/nginx.conf`. That configuration is unrelated to the application-facing edge proxy.

**Do not overwrite the Kubespray-generated `/etc/nginx/nginx.conf` or modify the existing `nginx-proxy` static Pods.** The external NGINX was isolated as its own containerized, host-networked workload instead. Running a host-installed `nginx.service` on the same 80/443 ports can cause conflicts.

## 7. Provisioning and Deployment Workflow

This section describes the lab workflow and verification checkpoints, **not a turnkey installation script**. Some manifest paths and Kubespray inventory directories depend on the local repository layout.

### Step 1 — Prepare the EC2 nodes

- Create four Ubuntu 24.04 EC2 VMs in the same VPC.
- Configure SSH access and an Ansible inventory containing both public management addresses and private Kubernetes node addresses.
- Configure AWS Security Groups so nodes can communicate inside the VPC.
- Confirm Ansible connectivity:

```bash
ansible -i inventory.ini all -m ping
```

### Step 2 — Provision Kubernetes using Kubespray

- Define `master01` and `master02` as control-plane nodes.
- Define `master01`, `master02`, and `worker01` as etcd members.
- Define `worker01` and `worker02` as workers.
- Apply the matching Kubespray inventory and cluster playbook from your Kubespray checkout.
- Copy the administrative kubeconfig from the control-plane node to the management machine and secure it.

When accessing the Kubernetes API using a changing public IP, the kubeconfig's `server` endpoint may need updating. In this lab, the API certificate was associated with a control-plane private address, so a matching `tls-server-name` setting was used. Do **not** disable certificate verification as a permanent workaround.

Verify:

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

### Step 3 — Install and configure the F5 NGINX Ingress Controller

The Ingress Controller was installed with Helm into namespace `nginx-system`, with IngressClass `nginx` and NodePort exposure for HTTP and HTTPS.

Verify the existing installation:

```bash
helm list -n nginx-system
kubectl get ingressclass
kubectl get svc -n nginx-system
kubectl get pods -n nginx-system -o wide
```

Check that its NodePort Service is exposing `30080` and `30443` before connecting the edge proxy.

### Step 4 — Run the external NGINX edge proxy

Because this AWS account could not provision a managed ELB/NLB, the lab uses a self-hosted NGINX TCP proxy deployed in `edge-system` and pinned to `worker02`, with `hostNetwork: true`.

Conceptual NGINX configuration:

```nginx
stream {
    upstream ingress_http {
        server 172.31.1.203:30080;
        server 172.31.44.17:30080;
    }
    upstream ingress_https {
        server 172.31.1.203:30443;
        server 172.31.44.17:30443;
    }
    server {
        listen 80;
        proxy_pass ingress_http;
    }
    server {
        listen 443;
        proxy_pass ingress_https;
    }
}
```

This is a **representative configuration**; use the actual ConfigMap and Deployment in your repository as the source of truth. Keep the worker private IPs in sync with the EC2 nodes.

Verify:

```bash
kubectl get pods -n edge-system -o wide
kubectl logs -n edge-system deployment/edge-nginx --tail=100
kubectl exec -n edge-system deployment/edge-nginx -- nginx -T
```

### Step 5 — Deploy the applications and Ingress

Deploy the React application and NGINX welcome application, each with an internal ClusterIP Service. Apply the Ingress rules:

```bash
kubectl apply -f k8s/ingress.yaml
kubectl get deployment,svc -n react-demo
kubectl describe ingress react-ingress -n react-demo
```

Example application manifests may be stored as `k8s/nginx-welcome.yaml` and separate React Deployment/Service files. Check your repository before running any path not present locally.

### Step 6 — Configure Cloudflare DNS

Create the two A records listed in [DNS and Application Routing](#5-dns-and-application-routing). Use **DNS only** while testing direct connectivity to the cluster.

```bash
dig +short k8s.kagea.site
dig +short k8s-nginx.kagea.site
```

Both should return the **current** `worker02` public IP.

### Step 7 — Install cert-manager and configure Let's Encrypt

cert-manager was installed using Helm in the `cert-manager` namespace. The `letsencrypt-prod` ClusterIssuer uses an ACME DNS-01 Cloudflare solver.

A scoped Cloudflare API token is stored as a Kubernetes Secret:

```text
Namespace: cert-manager
Secret:    cloudflare-api-token-secret
Key:       api-token
```

The token needs Cloudflare **Zone / DNS / Edit** and **Zone / Zone / Read** permissions, restricted to the `kagea.site` zone. Never commit the token or a Secret manifest containing its real value.

Apply the issuer and certificate resources (after cert-manager and the API token Secret exist):

```bash
kubectl apply -f k8s/letsencrypt-issuer.yaml
kubectl get clusterissuer letsencrypt-prod

kubectl apply -f k8s/certificate.yaml
kubectl get certificate -n react-demo
```

The `Certificate` resource is named `kagea-site-certificate`; it requests one certificate with both DNS names and writes it to `react-demo/kagea-site-tls`. The Ingress must reference **that same Secret**.

> A ClusterIssuer marked `Ready=True` means its ACME account is configured; it does not by itself prove Cloudflare DNS-01 validation works. Successful issuance of the `Certificate` is the stronger end-to-end test.

## 8. Verification and Test Results

### Cluster and workload checks

```bash
kubectl get nodes -o wide
kubectl get deployment,svc,pods -n react-demo
kubectl describe ingress react-ingress -n react-demo
kubectl get certificate,certificaterequest -n react-demo
kubectl get clusterissuer letsencrypt-prod
```

### Browser-trusted HTTPS: do not use `-k`

```bash
curl -Iv --connect-timeout 5 --max-time 15 https://k8s.kagea.site/
curl -Iv --connect-timeout 5 --max-time 15 https://k8s-nginx.kagea.site/
```

**Verified on 20 November 2026:**

| Check | Result |
| --- | --- |
| `https://k8s.kagea.site` | `HTTP/1.1 200 OK` |
| `https://k8s-nginx.kagea.site` | `HTTP/1.1 200 OK` |
| TLS protocol negotiated | TLS 1.3 |
| Certificate authority | Let's Encrypt (`YR2` issuer) |
| Certificate trust | `OpenSSL verify result: 0` |
| Certificate hostnames | Both `k8s.kagea.site` and `k8s-nginx.kagea.site` |
| Certificate valid from | 20 November 2026, 04:11:04 UTC |
| Certificate expires | 18 February 2027, 04:11:03 UTC |
| cert-manager scheduled renewal | 19 January 2027, 04:11:03 UTC |

Verify the actual certificate served by Ingress:

```bash
echo | openssl s_client \
  -connect k8s.kagea.site:443 \
  -servername k8s.kagea.site \
  2>/dev/null | openssl x509 -noout -issuer -dates -ext subjectAltName
```

Verify the certificate controller:

```bash
kubectl describe certificate kagea-site-certificate -n react-demo
```

### HTTP-to-HTTPS redirect: still needs verification

The latest public HTTP tests did not receive a response and were manually interrupted. Before declaring redirects working, test:

```bash
curl -4 -Iv --connect-timeout 5 --max-time 12 http://k8s.kagea.site/
curl -4 -Iv --connect-timeout 5 --max-time 12 http://k8s-nginx.kagea.site/
```

Once port 80 is reachable and the NGINX Ingress redirect annotation is enabled, the expected result is a `301` or another appropriate HTTP redirect to `https://...`.

If port 80 hangs, check in order:

1. The current `worker02` public IP and Cloudflare DNS records.
2. The Security Group attached to `worker02` permits inbound TCP `80`.
3. The edge proxy Pod is running and listening on port `80`.
4. The NGINX `stream` upstream forwards port `80` to the internal NodePort `30080`.
5. The Ingress Controller's HTTP NodePort Service is healthy.

Test the internal HTTP NodePort **from an EC2 node**:

```bash
curl -I --connect-timeout 5 --max-time 12 \
  -H 'Host: k8s.kagea.site' \
  http://172.31.44.17:30080/
```

## 9. Common Issues Encountered

| Symptom | Likely cause | Check / fix |
| --- | --- | --- |
| DNS points to old IP after EC2 restart | Public EC2 address changed | Refresh Cloudflare A records and inventory |
| Kubernetes API fails after restart | Kubeconfig uses stale public address or wrong certificate name | Update the endpoint and verify SAN / `tls-server-name` |
| HTTPS `SSL_ERROR_UNRECOGNIZED_NAME_ALERT` | TLS SNI hostname not configured or not recognized | Check Ingress hosts and TLS Secret; test with `curl --resolve` |
| Ingress warning: `nginx-service` not found | Missing Service in `react-demo` | Deploy NGINX workload and Service |
| Ingress warning: TLS Secret missing | Secret absent or in wrong namespace | Inspect `react-demo/kagea-site-tls` |
| HTTP `502 Bad Gateway` | Ingress upstream/backend missing or unavailable | Check Service selectors, endpoints, and Pods |
| Self-signed certificate trust warning | Temporary lab cert served instead of Let's Encrypt | Check issued Secret and Ingress `tls.secretName` |
| `ClusterIssuer Ready=False` just after creation | ACME account registration not finished, or registration error | `kubectl describe clusterissuer letsencrypt-prod` |
| `Certificate Ready=False` | ACME DNS-01 challenge not passing | Inspect Certificate, Orders, Challenges, Cloudflare token |
| HTTPS works, HTTP hangs | Public port 80 or TCP/NodePort forwarding problem | Diagnose SG, edge listener, and NodePort `30080` |
| OS NGINX fails on a worker | Collision/conflict with Kubespray-managed NGINX configuration | Keep the edge proxy isolated; do not overwrite static Pod config |

Useful cert-manager diagnostics:

```bash
kubectl describe certificate kagea-site-certificate -n react-demo
kubectl get orders,challenges -n react-demo
kubectl logs -n cert-manager deployment/cert-manager --tail=100
```

## 10. Security and Availability Notes

- **Allow only intended public ports.** Typically expose TCP `80`/`443` on the public edge; restrict SSH and Kubernetes API port `6443` to trusted management IPs.
- **Do not expose NodePorts unnecessarily.** Ports `30080`/`30443` are edge-to-cluster forwarding targets and can remain private.
- **Do not commit secrets.** Keep Cloudflare API tokens, kubeconfig admin credentials, TLS private keys, and `.env` files out of source control.
- **Prefer least-privilege Cloudflare API tokens.** Limit token permissions to the specific DNS zone.
- **Self-signed is for initial testing only.** The working configuration now uses a trusted Let's Encrypt certificate.
- **Certificate renewal needs a running cluster.** cert-manager schedules renewal automatically, but cannot perform it if the cluster remains shut down.
- **Public entry point is a single point of failure.** The edge proxy is pinned to `worker02`. Balancing its traffic across NodePorts does not make the edge tier itself highly available.
- **Control-plane availability is limited.** Although there are two control-plane nodes, etcd placement and the lack of an independent control-plane endpoint still require careful failure testing.
- **Cloudflare DNS-only does not protect or proxy origin HTTP traffic.** Adding Cloudflare proxying later would change the TLS/network trust model and should be evaluated separately.

## 11. Suggested Evidence / Screenshots

For a class presentation or project report, capture screenshots of:

1. AWS EC2 instances and their roles/private IPs (avoid showing account identifiers or private credentials).
2. `kubectl get nodes -o wide` showing four Ready nodes.
3. `kubectl get pods -A` showing control-plane, CNI, Ingress, and cert-manager components.
4. Helm release and NodePort Service in `nginx-system`.
5. External edge proxy Pod in `edge-system` and its NGINX `stream` configuration (no secrets).
6. Cloudflare DNS records for `k8s` and `k8s-nginx` (DNS only).
7. `kubectl describe ingress react-ingress -n react-demo` showing both host rules.
8. Both websites opening successfully over `https://` in a browser.
9. Browser certificate details showing Let's Encrypt and the covered DNS names.
10. `kubectl describe certificate kagea-site-certificate -n react-demo` showing `Ready=True` and `Renewal Time`.
11. HTTP redirect test once port `80` troubleshooting is complete.

## 12. Next Improvements

- [ ] Finish diagnosing public HTTP port 80 and confirm HTTP → HTTPS redirects.
- [ ] Automate Cloudflare DNS A-record updates when `worker02` public IP changes, or assign a stable public endpoint if available.
- [ ] Remove unnecessary public access to Kubernetes NodePorts.
- [ ] Add continuous deployment for the application (Jenkins/GitHub Actions and optionally Argo CD).
- [ ] Add observability (metrics, logs, alerts).
- [ ] Add application health checks, resource tuning, network policies, and rollout strategies.
- [ ] Revisit edge and control-plane high availability for a more resilient design.
- [ ] Add a standalone architecture diagram image to this README.

## References

- [Kubernetes documentation](https://kubernetes.io/docs/)
- [Kubespray](https://github.com/kubernetes-sigs/kubespray)
- [F5 NGINX Ingress Controller documentation](https://docs.nginx.com/nginx-ingress-controller/)
- [cert-manager documentation](https://cert-manager.io/docs/)
- [cert-manager Cloudflare DNS-01](https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/)
- [Let's Encrypt documentation](https://letsencrypt.org/docs/)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)

---

**Scope:** Educational DevOps / Kubernetes networking and TLS automation lab. Status and IP-related details reflect tests performed on **20 November 2026**, not a continuously monitored production service.
