# Homelab: k3d + Vault (Dev) + External Secrets Operator + Student API

This is a step-by-step guide for setting up a small Kubernetes homelab using:
- [k3d](https://k3d.io/) for the cluster
- [Vault](https://www.vaultproject.io/) in dev mode
- [External Secrets Operator](https://external-secrets.io/) to sync secrets
- Postgres + a Student API as the test app

---

## 1. Create the k3d Cluster

Expose the following:
- **80 / 443** → ingress (future use)
- **30080** → Student API
- **6550** → Kubernetes API

```bash
k3d cluster create homelab   --api-port 6550   --port "80:80@loadbalancer"   --port "443:443@loadbalancer"   --port "30080:30080@server:0"
```

Check the cluster:
```bash
kubectl get nodes
```

---

## 2. Install External Secrets Operator

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm install external-secrets external-secrets/external-secrets   --namespace external-secrets --create-namespace   --set installCRDs=true
```

Verify:
```bash
kubectl -n external-secrets get pods
```

---

## 3. Deploy Vault (Dev Mode)

```bash
kubectl create namespace vault
kubectl apply -f vault-deployment.yaml

Initialize VAULT
ubectl port-forward -n vault svc/vault 8200:8200 >/dev/null 2>&1 &
Then set the address:
export VAULT_ADDR=http://127.0.0.1:8200
vault kv put secret/student-api DB_USER=youruser DB_PASSWORD=yourpassword
```

---

## 4. Create SecretStore for ESO

```bash
kubectl -n vault create secret generic vault-token --from-literal=token=root
kubectl create namespace student-api
kubectl apply -f secretstore.yaml
```

---

## 5. Add Secrets to Vault

From inside the Vault pod:
```bash
kubectl -n vault exec -it deploy/vault -- sh

vault kv put secret/student-api DB_USER=postgres DB_PASS=postgres
vault kv put secret/db-credentials POSTGRES_USER=postgres POSTGRES_PASSWORD=postgres POSTGRES_DB=studentdb
exit
```

---

## 6. Create ExternalSecrets

```bash
kubectl apply -f externalsecrets.yaml
```

---

## 7. Deploy Postgres + Student API

```bash
kubectl -n student-api apply -f database.yaml
kubectl -n student-api apply -f student-api.yaml
```

---

With this setup, Vault and ESO manage your secrets, Postgres runs inside the cluster, and the Student API is reachable from your LAN on port `30080`.
