# K3S Bootstrap for FinBridge Services

## Stack
| Component     | Version         |
|---------------|-----------------|
| K3S           | v1.34.4+k3s1    |
| Helm          | v3.17.0         |
| MetalLB       | 0.14.9          |
| NGINX Ingress | 4.11.3          |
| cert-manager  | v1.16.3         |
| KEDA          | 2.16.0          |

MetalLB LB IP: **207.180.209.106** — point your DNS A records here.

---

## 1. Get kubeconfig from K3S server

```bash
# On the K3S server node
cat /etc/rancher/k3s/k3s.yaml
# Replace 127.0.0.1 with your server's public/private IP
```

## 2. Add KUBECONFIG_B64 to GitLab CI

```bash
# On your Mac
cat k3s.yaml | base64 | tr -d '\n'
# GitLab → Settings → CI/CD → Variables
# Name: KUBECONFIG_B64  |  Masked: true  |  Protected: true
```

## 3. Create namespaces

```bash
kubectl create namespace finbridge
kubectl create namespace finbridge-staging
```

## 4. GitLab registry pull secret (both namespaces)

```bash
for NS in finbridge finbridge-staging; do
  kubectl create secret docker-registry gitlab-registry-secret \
    --docker-server=registry.gitlab.com \
    --docker-username=<deploy-token-username> \
    --docker-password=<deploy-token-password> \
    -n $NS
done
```

## 5. Deploy ClusterIssuers (cert-manager — once per cluster)

```bash
# First deploy only — installs the Let's Encrypt ClusterIssuers
helm upgrade --install finbridge-service ./finbridge-service \
  --namespace finbridge \
  --set certManager.installIssuer=true \
  --set certManager.email=devops@bk.rw \
  --set image.tag=1.0.0 \
  --dry-run   # remove when ready

# All subsequent deploys omit certManager.installIssuer (defaults to false)
```

## 6. Create app secrets

```bash
kubectl create secret generic finbridge-service-secrets \
  --from-literal=DB_URL=jdbc:postgresql://postgres-host:5432/finbridge_db \
  --from-literal=DB_USERNAME=finbridge \
  --from-literal=DB_PASSWORD=<password> \
  --from-literal=JWT_SECRET=<your-jwt-secret> \
  -n finbridge

# Repeat for staging namespace
kubectl create secret generic finbridge-service-secrets \
  --from-literal=DB_URL=jdbc:postgresql://postgres-host:5432/finbridge_staging \
  --from-literal=DB_USERNAME=finbridge \
  --from-literal=DB_PASSWORD=<password> \
  --from-literal=JWT_SECRET=<your-jwt-secret> \
  -n finbridge-staging
```

## 7. DNS

Point your A records to MetalLB LB IP **207.180.209.106**:
```
finbridge.bk.rw         A  207.180.209.106
finbridge-staging.bk.rw A  207.180.209.106
```

cert-manager handles TLS automatically via HTTP-01 ACME challenge once DNS resolves.

## 8. Verify

```bash
# Pods
kubectl get pods -n finbridge

# Certificate issued by cert-manager
kubectl get certificate -n finbridge

# Ingress
kubectl get ingress -n finbridge

# KEDA ScaledObject (production)
kubectl get scaledobject -n finbridge

# Logs
kubectl logs -f deploy/finbridge-service -n finbridge
```

## 9. Staging uses letsencrypt-staging issuer

The staging values file uses `letsencrypt-staging` to avoid Let's Encrypt
rate limits (5 certs/week per domain). The cert will show as untrusted in
browsers but TLS still works. Switch to `letsencrypt-prod` once confirmed.
