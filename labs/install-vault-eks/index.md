# Install Vault on EKS
Amazon Elastic Kubernetes Service (EKS) can run and scale Vault in the Amazon Web Services (AWS) cloud or on-premises. Creating a Kubernetes cluster and launching Vault via the Helm chart can be accomplished all from the command-line.

In this tutorial, you create a cluster in AWS, deploy a MySQL server, install Vault in high-availability (HA) mode via the Helm chart and then configure the authentication between Vault and the cluster. Then you deploy a web application with deployment annotations so the application's secrets are installed via the Vault Agent injector service.

## Prerequisites
This tutorial requires: 
- AWS eksctl command-line tool 
- Kubernetes CLI tool `kubectl`
- Helm CLI 
- SSH key

Run the following commands: 

### Create a persistent directory for utilities
```sh
mkdir -p $HOME/.local/bin
cd $HOME/.local/bin
```

### Install kubectl
```sh
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.23.6/bin/linux/amd64/kubectl
chmod +x kubectl
```

### Install eksctl
Install the latest version of the `eksctl` tool
```sh
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl $HOME/.local/bin
```

### Install Helm 
```sh
export VERIFY_CHECKSUM=false
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
sudo mv /usr/local/bin/helm $HOME/.local/bin
```

### Generate SSH key
```sh
aws ec2 create-key-pair --key-name vault
```

Copy the key beginning at `-----BEGIN RSA PRIVATE KEY` and ending at `-----END RSA PRIVATE KEY-----"` and save to a file in CloudShell. 

### Create EKS cluster 
**NOTE: This can take up to 20 minutes**
```sh
eksctl create cluster \
    --name vault-demo \
    --nodes 3 \
    --with-oidc \
    --ssh-access \
    --ssh-public-key vault \
    --managed
```
The cluster is created, deployed and then health-checked. When the cluster is ready the command modifies the kubectl configuration so that the commands you issue are performed against that cluster.

Display the nodes of the cluster
```sh
kubectl get nodes
```

The cluster is ready.

### Install MySQL for Secrets backend 
MySQL is a fast, reliable, scalable, and easy to use open-source relational database system. MySQL Server is intended for mission-critical, heavy-load production systems as well as for embedding into mass-deployed software.

Add the Bitnami Helm repository.
```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Install the latest version of the MySQL Helm chart.
```sh
helm install mysql bitnami/mysql
```

By default the MySQL Helm chart deploys a single pod a service.

Get all the pods within the default namespace.
```sh
kubectl get pods
```
```
NAME                                    READY   STATUS    RESTARTS   AGE
mysql-0                                 1/1     Running   0          2m58s
```

Wait until the `mysql-0` pod is running and ready `(1/1)`.

Get all the services within the default namespace.
```sh
kubectl get services
```

```
kubernetes                 ClusterIP   10.100.0.1       <none>        443/TCP             3h24m
mysql                      ClusterIP   10.100.68.110    <none>        3306/TCP            15m
mysql-headless             ClusterIP   None             <none>        3306/TCP            15m
```

The MySQL root password is stored as Kubernetes secret. This password is required by Vault to create credentials for the application pod deployed later.

Create a variable named `ROOT_PASSWORD` that stores the mysql root user password.
```sh
ROOT_PASSWORD=$(kubectl get secret --namespace default mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode)
```

The MySQL server, addressed through the service, is ready.

### Install the Vault Helm chart
Add the HashiCorp Helm repository.
```sh
helm repo add hashicorp https://helm.releases.hashicorp.com
```

Update all the repositories to ensure `helm` is aware of the latest versions.
```sh
helm repo update
```

Search for all the Vault Helm chart versions.
```sh
helm search repo vault --versions
```
```
NAME            CHART VERSION   APP VERSION DESCRIPTION
hashicorp/vault 0.11.0          1.7.0       Official HashiCorp Vault Chart
## ...
```

The Vault Helm chart contains all the necessary components to run Vault in several different modes.

Install the latest version of the Vault Helm chart in HA mode with integrated storage.
```sh
helm install vault hashicorp/vault \
    --set='server.ha.enabled=true' \
    --set='server.ha.raft.enabled=true'
```

The Vault pods and Vault Agent Injector pod are deployed in the default namespace.

Get all the pods within the default namespace.
```sh
kubectl get pods
```

```
NAME                                    READY   STATUS    RESTARTS   AGE
vault-0                                 0/1     Running   0          30s
vault-1                                 0/1     Running   0          30s
vault-2                                 0/1     Running   0          30s
vault-agent-injector-56bf46695f-crqqn   1/1     Running   0          30s
```

The `vault-0`, `vault-1`, and `vault-2` pods deployed run a Vault server and report that they are `Running` but that they are not ready `(0/1)`. This is because the status check defined in a `readinessProbe` returns a non-zero exit code.

The `vault-agent-injector` pod deployed is a Kubernetes Mutation Webhook Controller. The controller intercepts pod events and applies mutations to the pod if specific annotations exist within the request.

Retrieve the status of Vault on the `vault-0` pod.
```sh
kubectl exec vault-0 -- vault status
```
```
Key                Value
---                -----
Seal Type          shamir
Initialized        false
Sealed             true
Total Shares       0
Threshold          0
Unseal Progress    0/0
Unseal Nonce       n/a
Version            1.7.0
Storage Type       raft
HA Enabled         true
command terminated with exit code 2
```

The status command reports that Vault is not initialized and that it is sealed. For Vault to authenticate with Kubernetes and manage secrets requires that that is initialized and unsealed.

### Initialize and unseal one Vault pod

Vault starts uninitialized and in the sealed state. Prior to initialization the integrated storage backend is not prepared to receive data.

Initialize Vault with one key share and one key threshold.
```sh
kubectl exec vault-0 -- vault operator init \
    -key-shares=1 \
    -key-threshold=1 \
    -format=json > cluster-keys.json

```

The `operator init` command generates a master key that it disassembles into key shares `-key-shares=1` and then sets the number of key shares required to unseal Vault `-key-threshold=1`. These key shares are written to the output as unseal keys in JSON format `-format=json`. Here the output is redirected to a file named `cluster-keys.json`.

Display the unseal key found in `cluster-keys.json`.
```sh
cat cluster-keys.json | jq -r ".unseal_keys_b64[]"
```
```
rrUtT32GztRy/pVWmcH0ZQLCCXon/TxCgi40FL1Zzus=
```

Create a variable named `VAULT_UNSEAL_KEY` to capture the Vault unseal key.
```sh
VAULT_UNSEAL_KEY=$(cat cluster-keys.json | jq -r ".unseal_keys_b64[]")
```

After initialization, Vault is configured to know where and how to access the storage, but does not know how to decrypt any of it. Unsealing is the process of constructing the master key necessary to read the decryption key to decrypt the data, allowing access to the Vault.


Unseal Vault running on the `vault-0` pod.
```sh
kubectl exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
```
```
Key                     Value
---                     -----
Seal Type               shamir
Initialized             true
Sealed                  false
Total Shares            1
Threshold               1
Version                 1.7.0
Storage Type            raft
Cluster Name            vault-cluster-3acfd011
Cluster ID              f1c8b22e-b569-341a-78ce-eb845ad4ba7e
HA Enabled              true
HA Cluster              n/a
HA Mode                 standby
Active Node Address     <none>
Raft Committed Index    24
Raft Applied Index      24
```

The `operator unseal` command reports that Vault is initialized and unsealed.

Retrieve the status of Vault on the `vault-0` pod.
```sh
kubectl exec vault-0 -- vault status
```
```
Key                     Value
---                     -----
Seal Type               shamir
Initialized             true
Sealed                  false
Total Shares            1
Threshold               1
Version                 1.7.0
Storage Type            raft
Cluster Name            vault-cluster-3acfd011
Cluster ID              f1c8b22e-b569-341a-78ce-eb845ad4ba7e
HA Enabled              true
HA Cluster              https://vault-0.vault-internal:8201
HA Mode                 active
Active Since            2021-04-01T16:12:51.9329527Z
Raft Committed Index    29
Raft Applied Index      29
```

The Vault server is initialized and unsealed.

### Join the other Vaults to the Vault cluster
The Vault server running on the `vault-0` pod is a Vault HA cluster with a single node. To display the list of nodes requires that you are logging in with the root token.

Display the root token found in `cluster-keys.json`.
```sh
cat cluster-keys.json | jq -r ".root_token"
```

Create a variable named `CLUSTER_ROOT_TOKEN` to capture the Vault unseal key.
```sh
CLUSTER_ROOT_TOKEN=$(cat cluster-keys.json | jq -r ".root_token")
```

Login with the root token on the `vault-0` pod.
```sh
kubectl exec vault-0 -- vault login $CLUSTER_ROOT_TOKEN
```

List all the nodes within the Vault cluster for the `vault-0` pod.
```sh
kubectl exec vault-0 -- vault operator raft list-peers
```

This displays the one node within the Vault cluster. This cluster is addressable through the Kubernetes service `vault-0.vault-internal` created by the Helm chart. The Vault servers on the other pods need to join this cluster and be unsealed.

Join the Vault server on `vault-1` to the Vault cluster.
```sh
kubectl exec vault-1 -- vault operator raft join http://vault-0.vault-internal:8200
```

This Vault server joins the cluster sealed. To unseal the Vault server requires the same unseal key, `VAULT_UNSEAL_KEY`, provided to the first Vault server.

Unseal the Vault server on `vault-1` with the unseal key.

The Vault server on `vault-1` is now a functional node within the Vault cluster.

Join the Vault server on `vault-2` to the Vault cluster.
```sh
kubectl exec vault-2 -- vault operator raft join http://vault-0.vault-internal:8200
```

Unseal the Vault server on `vault-2` with the unseal key.
```sh
kubectl exec vault-2 -- vault operator unseal $VAULT_UNSEAL_KEY
```

The Vault server on `vault-2` is now a functional node within the Vault cluster.

List all the nodes within the Vault cluster for the `vault-0` pod.
```sh
kubectl exec vault-0 -- vault operator raft list-peers
```

This displays all three nodes within the Vault cluster.

List pods in the `default` namespace
```sh
kubectl get pods
```

All pods should report that they are Running and ready (1/1).

### Create a Vault database role
The web application that you deploy in the Launch a web application section, expects Vault to store a username and password at the path secret/webapp/config. To create this secret requires you to login with the root token, enable the key-value secret engine, and store a secret username and password at that defined path.

Enable database secrets at the path `database`.
```sh
kubectl exec vault-0 -- vault secrets enable database
```   
```sh
Success! Enabled the database secrets engine at: database/
```

Configure the database secrets engine with the connection credentials for the MySQL database.
{% raw %}
```sh
kubectl exec vault-0 -- vault write database/config/mysql \
    plugin_name=mysql-database-plugin \
    connection_url="{{username}}:{{password}}@tcp(mysql.default.svc.cluster.local:3306)/" \
    allowed_roles="readonly" \
    username="root" \
    password="$ROOT_PASSWORD"
```
{% endraw %}


Create a database secrets engine role named `readonly`.
```sh
kubectl exec vault-0 -- vault write database/roles/readonly \
    db_name=mysql \
    creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO '{{name}}'@'%';" \
    default_ttl="1h" \
    max_ttl="24h"
```    

The readonly role generates credentials that are able to perform queries for any table in the database.

Read credentials from the readonly database role.
```sh
kubectl exec vault-0 -- vault read database/creds/readonly
```
```
Key                Value
---                -----
lease_id           database/creds/readonly/qtWlgBT1YTQEPKiXe7CrotsT
lease_duration     1h
lease_renewable    true
password           WLESe5T-RLkTj-h-lDbT
username           v-root-readonly-pk168KvLS8sc80Of
```

Vault is able to generate credentials within the MySQL database.

### Configure Vault Kubernetes authentication
The initial root token is a privileged user that can perform any operation at any path. The web application only requires the ability to read secrets defined at a single path. This application should authenticate and be granted a token with limited access.

Vault provides a Kubernetes authentication method that enables clients to authenticate with a Kubernetes Service Account Token.

First, start an interactive shell session on the `vault-0` pod.
```sh
kubectl exec --stdin=true --tty=true vault-0 -- /bin/sh
```

Your system prompt is replaced with a new prompt `/ $`.

**NOTE: the prompt within this section is shown as $ but the commands are intended to be executed within this interactive shell on the vault-0 container.**

Enable the Kubernetes authentication method.
```sh
vault auth enable kubernetes
```

Vault accepts this service token from any client within the Kubernetes cluster. During authentication, Vault verifies that the service account token is valid by querying a configured Kubernetes endpoint.

Configure the Kubernetes authentication method to use the service account token, the location of the Kubernetes host, and its certificate.

```sh
vault write auth/kubernetes/config \
      token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
      kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
      kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```
The `token_reviewer_jwt` and `kubernetes_ca_cert` are mounted to the container by Kubernetes when it is created. The environment variable `KUBERNETES_PORT_443_TCP_ADDR` is defined and references the internal network address of the Kubernetes host.

For a client of the Vault server to read the credentials requires that the read capability be granted for the path `database/creds/readonly`.
Write out the policy named `devwebapp` that enables the `read` capability for secrets at path `database/creds/readonly`.

Exit the `vault-0` pod 
```sh
exit
```

```sh
vault policy write devwebapp - <<EOF
path "database/creds/readonly" {
  capabilities = ["read"]
}
EOF
```

Create a Kubernetes authentication role named `devweb-app`.
```sh
vault write auth/kubernetes/role/devweb-app \
      bound_service_account_names=internal-app \
      bound_service_account_namespaces=default \
      policies=devwebapp \
      ttl=24h
```
The role connects a Kubernetes service account, `internal-app` (created in the next step), and namespace, `default`, with the Vault policy, `devwebapp`. The tokens returned after authentication are valid for 24 hours.

Lastly, exit the `vault-0` pod.

```sh
exit
```

### Deploy Web App
The web application pod requires the creation of the `internal-app` Kubernetes service account.

Define a Kubernetes service account named `internal-app`.
```sh
cat > internal-app.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: internal-app
EOF
```

Create the `internal-app` service account.
```sh
kubectl apply --filename internal-app.yaml
```

Define a pod named `devwebapp` with the web application.
{% raw %}
```sh
cat > devwebapp.yaml <<EOF
---
apiVersion: v1
kind: Pod
metadata:
  name: devwebapp
  labels:
    app: devwebapp
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/agent-cache-enable: "true"
    vault.hashicorp.com/role: "devweb-app"
    vault.hashicorp.com/agent-inject-secret-database-connect.sh: "database/creds/readonly"
    vault.hashicorp.com/agent-inject-template-database-connect.sh: |
      {{- with secret "database/creds/readonly" -}}
      mysql -h my-release-mysql.default.svc.cluster.local --user={{ .Data.username }} --password={{ .Data.password }} my_database

      {{- end -}}
spec:
  serviceAccountName: internal-app
  containers:
    - name: devwebapp
      image: jweissig/app:0.0.1
EOF
```
{% endraw %}

Create the `devwebapp` pod.
```sh
kubectl apply -f devwebapp.yaml
```

This definition creates a pod with the specified container running with the `internal-app` Kubernetes service account. The container within the pod is unaware of the Vault cluster. The Vault Injector service reads the annotations and determines that it should take action `vault.hashicorp.com/agent-inject`. The credentials, read from Vault at `database/creds/readonly`, are retrieved by the `devwebapp-role` Vault role and stored at the file location, `/vault/secrets/database-connect.sh`, and then mounted in the pod.

The credentials are requested first by the vault-agent-init container to ensure they are present when the application pod initializes. After the application pod initializes, the injector service creates a vault-agent pod that assists the application in maintaining the credentials during initialization. The credentials requested by the vault-agent-init container are cached, `vault.hashicorp.com/agent-cache-enable: "true"`, and used by vault-agent container.

List running pods 
```sh
kubectl get pods
```
```
devwebapp                               2/2     Running   0          6s
mysql-0                                 1/1     Running   0          12m
vault-0                                 1/1     Running   0          8m48s
vault-1                                 1/1     Running   0          8m48s
vault-2                                 1/1     Running   0          8m48s
vault-agent-injector-56bf46695f-gdpsz   1/1     Running   0          8m48s
```

Wait until the `devwebapp` pod reports that is running and ready `(2/2)`.

Display the secrets written to the file `/vault/secrets/database-connect.sh` on the `devwebapp` pod.
```sh
kubectl exec --stdin=true \
    --tty=true devwebapp \
    --container devwebapp \
    -- cat /vault/secrets/database-connect.sh
```

The result displays a `mysql` command with the credentials generated for this pod.
```
mysql -h my-release-mysql.default.svc.cluster.local --user=v-kubernetes-readonly-zpqRzAee2b --password=Jb4epAXSirS2s-pnrI9- my_database
```

# Congrats!

