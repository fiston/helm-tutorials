# K3S Bootstrap for FinBridge Services

## 1. Get your kubeconfig from K3S

```bash
# On the K3S server
cat /etc/rancher/k3s/k3s.yaml
# Replace 127.0.0.1 with your server's actual IP
```

## 2. Add KUBECONFIG_B64 to GitLab CI variables

```bash
cat k3s.yaml | base64 | tr -d '\n'
# Paste the output into GitLab → Settings → CI/CD → Variables
# Variable name: KUBECONFIG_B64
# Masked: true, Protected: true
```

## 3. Create namespaces

```bash
kubectl create namespace finbridge
kubectl create namespace finbridge-staging
```

## 4. Create the GitLab registry pull secret in each namespace

```bash
kubectl create secret docker-registry gitlab-registry-secret \
  --docker-server=registry.gitlab.com \
  --docker-username=<gitlab-deploy-token-username> \
  --docker-password=<gitlab-deploy-token-password> \
  -n finbridge

kubectl create secret docker-registry gitlab-registry-secret \
  --docker-server=registry.gitlab.com \
  --docker-username=<gitlab-deploy-token-username> \
  --docker-password=<gitlab-deploy-token-password> \
  -n finbridge-staging
```

## 5. Create app secrets

```bash
kubectl create secret generic finbridge-service-secrets \
  --from-literal=DB_URL=jdbc:postgresql://postgres-host:5432/finbridge_db \
  --from-literal=DB_USERNAME=finbridge \
  --from-literal=DB_PASSWORD=<password> \
  --from-literal=JWT_SECRET=<your-jwt-secret> \
  -n finbridge
```

## 6. Manual Helm deploy (first time or debugging)

```bash
helm upgrade --install finbridge-service ./helm/finbridge-service \
  --namespace finbridge \
  --values ./helm/finbridge-service/values.yaml \
  --values ./helm/finbridge-service/values-production.yaml \
  --set image.tag=1.0.0 \
  --dry-run   # remove this when ready
```

## 7. Verify

```bash
kubectl get pods -n finbridge
kubectl logs -f deploy/finbridge-service -n finbridge
kubectl get ingress -n finbridge
```
