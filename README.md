# Creating a Cluster

`sudo k3d cluster create --k3s-arg "--tls-san=${IP_ADDRESS}@server:0" --port 80:80@loadbalancer --port 443:443@loadbalancer --api-port 6443`

# Bootstrapping with this repo

` export GITHUB_TOKEN=PLACE_TOKEN_HERE`

Note the space at the beginning of that one...  If you setup your shell right, that space will prevent the token from going into your history.

`sudo --preserve-env=GITHUB_TOKEN flux bootstrap github --owner=kc0bfv --repository=trying_flux_infrastructure --path=clusters/DESIRED_FOLDER --personal`

# Coder Secrets

To install coder you'll need to create a database key before installing it.  If you fail to do so - you'll have to delete the DB PVC before uninstalling the chart and letting it all recreate with the right password.

So - create the secrets in advance.

```
 export DBPASSWD=MakeSomethingNiceUpHere
sudo kubectl create namespace coder

sudo kubectl create secret generic coder-db-creds -n coder --from-literal="password=${DBPASSWD}" --from-literal="postgres-password=${DBPASSWD}" --from-literal="replication-password=${DBPASSWD}"
sudo kubectl create secret generic coder-db-url -n coder --from-literal=url="postgres://codea:${DBPASSWD}@coder-db-postgresql.coder.svc.cluster.local:5432/coder?sslmode=disable"
```
