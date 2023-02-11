# Creating a Cluster

```
sudo mkdir /root/.k3d-cache

sudo k3d cluster create --k3s-arg "--tls-san=${IP_ADDRESS}@server:0" --port 80:80@loadbalancer --port 443:443@loadbalancer --api-port 6443 --volume /root/.k3d-cache:/var/lib/rancher/k3s/
```

# Bootstrapping with this repo

` export GITHUB_TOKEN=PLACE_TOKEN_HERE`

Note the space at the beginning of that one...  If you setup your shell right, that space will prevent the token from going into your history.

```
export CLUSTER="local_test"

sudo --preserve-env=GITHUB_TOKEN flux bootstrap github --owner=kc0bfv --repository=trying_flux_infrastructure --path=clusters/${CLUSTER} --personal
```

# Secrets

Based on https://fluxcd.io/flux/guides/sealed-secrets/

Install kubeseal from [GitHub](https://github.com/bitnami-labs/sealed-secrets/releases)

After bootstrapping on deployed machine, run:

```
sudo kubeseal --fetch-cert --controller-name=sealed-secrets-controller --controller-namespace=flux-system > public_sealed_secret.pem
```

Create your secrets with `-o yaml --dry-run=client` tacked onto the kubectl command, and save the output into files.

For coder:

```
 export DBPASSWD=MakeSomethingNiceUpHere

kubectl create secret generic coder-db-creds -n coder --from-literal="password=${DBPASSWD}" --from-literal="postgres-password=${DBPASSWD}" --from-literal="replication-password=${DBPASSWD}" -o yaml --dry-run=client > coder-db-creds.yaml
kubectl create secret generic coder-db-url -n coder --from-literal=url="postgres://coder:${DBPASSWD}@coder-db-postgresql.coder.svc.cluster.local:5432/coder?sslmode=disable" -o yaml --dry-run=client > coder-db-url.yaml
```

For power sensor monitor:

```
 export WRITEKEY=MakeSomethingNiceUpHere
 export READKEY=MakeSomethingNiceUpHere

kubectl create secret generic webhook-catcher-keys -n power-sensor-monitor --from-literal="writekey=${WRITEKEY}" --from-literal="readkey=${READKEY}" -o yaml --dry-run=client > webhook-catcher-keys.yaml
```

Encrypt the secrets:

```
kubeseal --format=yaml --cert=public_sealed_secret.pem < coder-db-creds.yaml > coder-db-creds-sealed.yaml
kubeseal --format=yaml --cert=public_sealed_secret.pem < coder-db-url.yaml > coder-db-url-sealed.yaml
kubeseal --format=yaml --cert=public_sealed_secret.pem < webhook-catcher-keys.yaml > webhook-catcher-keys-sealed.yaml
```

Delete the unencrypted secrets, and the public key if you want:

```
rm coder-db-creds.yaml coder-db-url.yaml webhook-catcher-keys.yaml public_sealed_secret.pem
```

Move the sealed secrets back to your git repo development box, and drop them in the `secrets/TARGET_CLUSTER` folder.  Commit that and they'll flow to the target cluster.
