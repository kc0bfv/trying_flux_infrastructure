# Creating a Cluster With K3S

```sh
export EXT_IP_ADDRESS=Put external IP address here
export WG_IP_ADDRESS=Put wireguard IP address here
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--tls-san=${EXT_IP_ADDRESS} --tls-san=${WG_IP_ADDRESS} --advertise-address=${WG_IP_ADDRESS} --node-ip=${WG_IP_ADDRESS} --kubelet-arg=provider-id=aws:///us-east-2a/i-doesnotexist" sh -
```

As root in root's home dir...
```sh
mkdir .kube
ln -s /etc/rancher/k3s/k3s.yaml .kube/config
```

You will want to add some firewall rules...

```
-A INPUT -p tcp -m tcp --dport 22 -j ACCEPT
-A INPUT -p tcp -m tcp --dport 80 -j ACCEPT
-A INPUT -p tcp -m tcp --dport 443 -j ACCEPT
-A INPUT -i wg0 -p tcp -m tcp --dport 6443 -j ACCEPT
-A INPUT -p tcp -m conntrack --ctstate NEW -j DROP
```

# Adding another External Node to the Cluster

1. Setup wireguard to point-to-point vpn your cluster nodes together.
2. Install k3s on the external node and connect it back.

## Setup Wireguard Point-to-Point

https://www.digitalocean.com/community/tutorials/how-to-create-a-point-to-point-vpn-with-wireguard-on-ubuntu-16-04

Setup a wireguard key on all nodes.

```sh
(umask 077 && printf "[Interface]\nPrivateKey = " | sudo tee /etc/wireguard/wg0.conf > /dev/null)
wg genkey | sudo tee -a /etc/wireguard/wg0.conf | wg pubkey | sudo tee /etc/wireguard/publickey
```

Add the desired private network host addresses to each `/etc/wireguard/wg0.conf`.

```
[Interface]
PrivateKey = ...
ListenPort = 51820
Address = 10.[desired_net1].[desired_net2].[host_addr]/24
```

Add each peer to `wg0.conf` - each node must have the others listed as peers.

```
[Peer]
PublicKey = [remote public key goes here]
AllowedIPs = [network from interface above goes here]/24
Endpoint = [remote public address goes here]:51820
```

Start wireguard, make sure it works, then enable it at boot.

```
sudo systemctl restart wg-quick@wg0
sudo systemctl enable wg-quick@wg0
```

## Install k3s on the Remote Node

Now the nodes have a method to communicate securely, and IP addresses that can remain stable.

Get the token from the k3s master (run as root): `cat /var/lib/rancher/k3s/server/node-token`

On the new host - note the spaces at the beginning of some lines to prevent things going into shell history:

```sh
 WIREGUARD_HOST=put the host address here
 TOKEN=put the token here
curl -sfL https://get.k3s.io | K3S_URL="https://${WIREGUARD_HOST}:6443" K3S_TOKEN="${TOKEN}" sh -
```

# Bootstrapping with this repo

Note the space at the beginning of some of these...  If you setup your shell right, that space will prevent the token from going into your history.

```
 export GITHUB_TOKEN=PLACE_TOKEN_HERE
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

Create your secrets with `-o yaml --dry-run=client` tacked onto the kubectl command, pipe it to kubeseal to encrypt, and save the output into files.

For coder:

```
 export DBPASSWD=MakeSomethingNiceUpHere

kubectl create secret generic coder-db-creds -n coder --from-literal="password=${DBPASSWD}" --from-literal="postgres-password=${DBPASSWD}" --from-literal="replication-password=${DBPASSWD}" -o yaml --dry-run=client | kubeseal --format=yaml --cert=public_sealed_secret.pem > coder-db-creds-sealed.yaml
kubectl create secret generic coder-db-url -n coder --from-literal=url="postgres://coder:${DBPASSWD}@coder-db-postgresql.coder.svc.cluster.local:5432/coder?sslmode=disable" -o yaml --dry-run=client | kubeseal --format=yaml --cert=public_sealed_secret.pem > coder-db-url-sealed.yaml
```

For power sensor monitor:

```
 export WRITEKEY=MakeSomethingNiceUpHere
 export READKEY=MakeSomethingNiceUpHere

kubectl create secret generic power-sensor-monitor -n power-sensor-monitor --from-literal="writekey=${WRITEKEY}" --from-literal="readkey=${READKEY}" -o yaml --dry-run=client | kubeseal --format=yaml --cert=public_sealed_secret.pem > power-sensor-monitor-sealed.yaml
```

For ttrss:

```
 export ADMIN_PASS=MakeSomethingNiceUpHere
 export DB_PASS=MakeSomethingNiceUpHere
 export BASIC_AUTH_USER=MakeSomethingNiceUpHere
 export BASIC_AUTH_PASS=MakeSomethingNiceUpHere

kubectl create secret generic ttrss-database -n ttrss --from-literal="admin-pass=${ADMIN_PASS}" --from-literal="password=${DB_PASS}" -o yaml --dry-run=client | kubeseal --format=yaml --cert=public_sealed_secret.pem > ttrss-database-sealed.yaml
kubectl create secret generic ttrss-basic-auth -n ttrss --from-literal=username="${BASIC_AUTH_USER}" --from-literal=password="${BASIC_AUTH_PASS}" --type="kubernetes.io/basic-auth" -o yaml --dry-run=client | kubeseal --format=yaml --cert=public_sealed_secret.pem > ttrss-basic-auth-sealed.yaml
```

For Funkwhale:
```
 export DJANGO_SECRET_KEY=
 export AWS_ACCESS_KEY_ID=
 export AWS_SECRET_ACCESS_KEY=
 export AWS_STORAGE_BUCKET_NAME=
 export AWS_S3_REGION_NAME=

kubectl create secret generic funkwhale-separate-secret -n funkwhale --from-literal="DJANGO_SECRET_KEY=${DJANGO_SECRET_KEY}" --from-literal="AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}" --from-literal="AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}" --from-literal="AWS_STORAGE_BUCKET_NAME=${AWS_STORAGE_BUCKET_NAME}" --from-literal="AWS_S3_REGION_NAME=${AWS_S3_REGION_NAME}" -o yaml --dry-run=client | kubeseal --format=yaml --cert=public_sealed_secret.pem > funkwhale-separate-secret-sealed.yaml
```

For Funkwhale backup:
```
 export AWS_ACCESS_KEY_ID=
 export AWS_SECRET_ACCESS_KEY=
 export AWS_STORAGE_BUCKET_NAME=
 export DATABASE_USER=
 export DATABASE_PASS=
kubectl create secret generic backup-funkwhale -n funkwhale --from-literal="aws-access-key-id=${AWS_ACCESS_KEY_ID} --from-literal="aws-secret-access-key=${AWS_SECRET_ACCESS_KEY}" --from-literal="aws-storage-bucket=${AWS_STORAGE_BUCKET_NAME}" --from-literal="database-user=${DATABASE_USER}" --from-literal="database-password=${DATABASE_PASS}" -o yaml --dry-run=client | kubeseal --format=yaml --cert=public_sealed_secret.pem > backup-funkwhale-sealed.yaml
```

For AWS creds:
```
  export AWS_ACCESS_KEY=...
  export AWS_SECRET_ACCESS_KEY=...

kubectl create secret generic aws-secret -n kube-system --from-literal="aws_access_key_id=${AWS_ACCESS_KEY}" --from-literal="aws_secret_access_key=${AWS_SECRET_ACCESS_KEY}" -o yaml --dry-run=client | kubeseal --format=yaml --cert=public_sealed_secret.pem > aws-creds-sealed.yaml
```

For syncthing:

```
 export BASIC_AUTH_USER=MakeSomethingNiceUpHere
 export BASIC_AUTH_PASS=MakeSomethingNiceUpHere

kubectl create secret generic syncthing-basic-auth -n syncthing --from-literal=username="${BASIC_AUTH_USER}" --from-literal=password="${BASIC_AUTH_PASS}" --type="kubernetes.io/basic-auth" -o yaml --dry-run=client | kubeseal --format=yaml --cert=public_sealed_secret.pem > syncthing-basic-auth-sealed.yaml
```

Move the sealed secrets back to your git repo development box, and drop them in the `secrets/TARGET_CLUSTER` folder.  Commit that and they'll flow to the target cluster.  Run `sudo flux reconcile source git flux-system` to speed up the flow.
