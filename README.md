# rails-vault

This application demonstrates and approach to using Vault secrets to connect a Rails application to a Postgresql database.  Perform the following steps to install the example on your Kubernetes cluster choice.




Install Postgres:
```
helm install books-db oci://registry-1.docker.io/bitnamicharts/postgresql -f ./postgresql/values.yaml
```

Install Vault:
```
helm upgrade vault hashicorp/vault -f ./vault/values.yaml --install
```

Open a shell into the Vault pod.
```
kubectl exec -ti vault-0 -- /bin/sh
```

Create a new Policy.
```
cat <<EOF > /home/vault/app-policy.hcl
path "secret*" {
    capabilities = ["read"]
}
EOF
```

Save the Policy
```
vault policy write app /home/vault/app-policy.hcl
```

Enable Kubernetes security so that we can secure certain keys to certain Service Accounts.
```
vault auth enable kubernetes
```

Configure Kubernetes security.
```
vault write auth/kubernetes/config \
   token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
   kubernetes_host=https://${KUBERNETES_PORT_443_TCP_ADDR}:443 \
   kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

Grant permissions to the rails-vault Service Account using the app policy.
```
vault write auth/kubernetes/role/myapp \
   bound_service_account_names=rails-vault \
   bound_service_account_namespaces=default \
   policies=app \
   ttl=1h
```

Add some values to the secret Vault namespace.
```
vault kv put secret/books_database username=books password=secret
```

Build the rails-vault container images.
```
bundle install
bundle exec rails assets:precompile
docker build -t rails-vault:0.0.1 .
```

Install the rails-vault application.
```
helm install rails-vault ./helm
```

Check the value injected into the Pod.
```
export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=rails-vault,app.kubernetes.io/instance=rails-vault" -o jsonpath="{.items[0].metadata.name}")
kubectl exec -ti $POD_NAME -c rails-vault -- cat /vault/secrets/books_database
```

In a separate terminal, start a port forward:
```
kubectl port-forward deployment/rails-vault 3000:3000
```

Open http://localhost:3000 in a browser.