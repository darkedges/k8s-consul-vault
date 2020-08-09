# Hashicorp

## Metallb

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml
# On first install only
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
kubectl apply -f metallb.yaml
```

## Consul

### Deploy Consul

```bash
helm install consul hashicorp/consul -f consul-values.yaml
```

### Configure Ingress Gateway

```bash - ingress-gateway.hcl
Kind = "ingress-gateway"
Name = "ingress-gateway"

Listeners = [
 {
   Port = 8080
   Protocol = "http"
   Services = [
     {
       Name = "dashboard"
       Hosts = ["dashboard.default.k8s.darkedges.com"]
     },
     {
       Name = "static-server"
       Hosts = ["static.default.k8s.darkedges.com"]
     },
     {
       Name = "app"
       Hosts = ["webapp.default.k8s.darkedges.com"]
     },
   ]
 }
]
```

```bash
kubectl exec -it consul-9ftnd /bin/s
consul config write ingress-gateway.hcl
```

## Vault

### Deploy Vault

```bash
helm install vault hashicorp/vault -f vault-values.yaml
kubectl exec -ti vault-0 -- vault operator init -format=json > cluster-keys.json
```

returns  

```bash
Unseal Key 1: X1txfU7YL3OpLXKY6oWXqQqALnyXxOgcc8vgEMkgAb2f
Unseal Key 2: xixC7b7/zNW7ZLY6igHUb0vQ84O9CTZ3OCEMtED2KB1z
Unseal Key 3: n/tTp2uPfdhz9kSOpPWW8O/kybjRBqLf73K9ACHfR+tg
Unseal Key 4: udnb1A7ha0TloeClSqc0Ngkx0BlSy51wAEVTHJ9364/s
Unseal Key 5: 4W3D+MpF7xLA5XX2Bsg8eS6qwTp/56aZHFr9DAvaI9j7

Initial Root Token: s.vL6B1kwJKdQkoFr8rOpHAIMz

Vault initialized with 5 key shares and a key threshold of 3. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 3 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated master key. Without at least 3 key to
reconstruct the master key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.
```

```bash
kubectl exec -ti vault-0 -- vault operator unseal X1txfU7YL3OpLXKY6oWXqQqALnyXxOgcc8vgEMkgAb2f
kubectl exec -ti vault-0 -- vault operator unseal xixC7b7/zNW7ZLY6igHUb0vQ84O9CTZ3OCEMtED2KB1z
kubectl exec -ti vault-0 -- vault operator unseal n/tTp2uPfdhz9kSOpPWW8O/kybjRBqLf73K9ACHfR+tg
kubectl exec -ti vault-1 -- vault operator unseal X1txfU7YL3OpLXKY6oWXqQqALnyXxOgcc8vgEMkgAb2f
kubectl exec -ti vault-1 -- vault operator unseal xixC7b7/zNW7ZLY6igHUb0vQ84O9CTZ3OCEMtED2KB1z
kubectl exec -ti vault-1 -- vault operator unseal n/tTp2uPfdhz9kSOpPWW8O/kybjRBqLf73K9ACHfR+tg
kubectl exec -ti vault-2 -- vault operator unseal X1txfU7YL3OpLXKY6oWXqQqALnyXxOgcc8vgEMkgAb2f
kubectl exec -ti vault-2 -- vault operator unseal xixC7b7/zNW7ZLY6igHUb0vQ84O9CTZ3OCEMtED2KB1z
kubectl exec -ti vault-2 -- vault operator unseal n/tTp2uPfdhz9kSOpPWW8O/kybjRBqLf73K9ACHfR+tg
```

## Cet Manager

```bash
helm install cert-manager jetstack/cert-manager --namespace cert-manager --version v0.16.1 --set installCRDs=true
```

### Error 1

$ kubectl logs counting -c consul-connect-inject-init
Error registering service "counting-sidecar-proxy": Unexpected response code: 500 (could not retrieve initial service_defaults config for service "counting-counting-sidecar-proxy": No known Consul servers)

had to restart consul-xxxxx pod.

### Error 2

/tmp # consul config write ingress-gateway.hcl
Error writing config entry ingress-gateway/ingress-gateway: Unexpected response code: 500 (rpc error making call: rpc error making call: service "dashboard" has protocol "tcp", which does not match defined listener protocol "http")

need to add an annotation 
'consul.hashicorp.com/connect-service-protocol': 'http'
to the pods being deployed

## Vault Example App

```bash
kubectl exec -it vault-0 -- /bin/sh

vault login
vault secrets enable -path=secret kv-v2
vault kv put secret/webapp/config username="static-user" password="static-password"

vault auth enable kubernetes
vault write auth/kubernetes/config \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
vault policy write webapp - <<EOF
path "secret/data/webapp/config" {
  capabilities = ["read"]
}
EOF
vault write auth/kubernetes/role/webapp \
  bound_service_account_names=vault \
  bound_service_account_namespaces=default \
  policies=webapp \
  ttl=24h

kubectl apply --filename vault-example-app.yml
```
