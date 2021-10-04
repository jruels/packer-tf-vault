# Install Vault   

## Overview
Google Kubernetes Engine (GKE) can run Vault in its secured and managed Kubernetes service. Creating a GKE cluster and launching Vault via the Helm chart can be accomplished all from the command-line.

In this tutorial, you create a cluster in GKE, install Vault in high-availability (HA) mode via the Helm chart and then configure the authentication between Vault and the cluster. Then you deploy a web application with deployment annotations so the application's secrets are installed via the Vault Agent injector service.

## Pre-Reqs
Set a default compute zone
```
gcloud config set compute/zone us-west1-a
```

## Start GKE cluster
For a project to create a Kubernetes cluster requires that the container service is enabled.

Enable the Google container service.
```
gcloud services enable container.googleapis.com
```

A Vault cluster that is launched in high-availability requires a Kubernetes cluster with three nodes.

Create a cluster named `learn-vault` with 3 nodes.
```
gcloud container clusters create learn-vault --num-nodes=3
```

The cluster is created, deployed and then health-checked. When the cluster is ready the command modifies the `kubectl` configuration so that the commands you issue are performed against that cluster.

Display the nodes of the cluster.
```
kubectl get nodes
```

Output:
```
NAME                                         STATUS   ROLES    AGE   VERSION
gke-learn-vault-default-pool-bd170fcf-0vqg   Ready    <none>   19m   v1.16.15-gke.4300
gke-learn-vault-default-pool-bd170fcf-lrwj   Ready    <none>   19m   v1.16.15-gke.4300
gke-learn-vault-default-pool-bd170fcf-zn6k   Ready    <none>   19m   v1.16.15-gke.4300
```

The cluster is ready.


## Install the Vault Helm chart
The recommended way to run Vault on Kubernetes is via the Helm chart.

Add the HashiCorp Helm repository.
```
helm repo add hashicorp https://helm.releases.hashicorp.com
```

Update all the repositories to ensure `helm` is aware of the latest versions.

```
helm repo update
```

Search for all the Vault Helm chart versions. 
```
helm search repo vault --versions
```

Output:
```
NAME            CHART VERSION   APP VERSION DESCRIPTION
hashicorp/vault 0.11.0           1.7.0       Official HashiCorp Vault Chart
...
```

The Vault Helm chart contains all the necessary components to run Vault in several different modes.

Install the latest version of the Vault Helm chart in HA mode with integrated storage.
```
helm install vault hashicorp/vault \
  --set='server.ha.enabled=true' \
  --set='server.ha.raft.enabled=true'
```

The Vault pods and Vault Agent Injector pod are deployed in the default namespace.

Get all the pods within the default namespace.
```
kubectl get pods 
```

The `vault-0`, `vault-1`, and `vault-2` pods deployed run a Vault server and report that they are Running but that they are not ready (0/1). This is because the status check defined in a readinessProbe returns a non-zero exit code.

The `vault-agent-injector` pod deployed is a Kubernetes Mutation Webhook Controller. The controller intercepts pod events and applies mutations to the pod if specific annotations exist within the request.

Retrieve the status of Vault on the `vault-0` pod.

```
kubectl exec vault-0 -- vault status
```

Output:
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
Version            n/a
HA Enabled         false
command terminated with exit code 2
```

The status command reports that Vault is not initialized and that it is sealed. For Vault to authenticate with Kubernetes and manage secrets requires that that is initialized and unsealed.

## Initialize and unseal one Vault pod

Vault starts uninitialized and in the sealed state. Prior to initialization the integrated storage backend is not prepared to receive data.

Initialize Vault with one key share and one key threshold.

```
kubectl exec vault-0 -- vault operator init -key-shares=1 -key-threshold=1 -format=json > cluster-keys.json
```

The `operator init` command generates a master key that it disassembles into key shares `-key-shares=1` and then sets the number of key shares required to unseal Vault `-key-threshold=1`. These key shares are written to the output as unseal keys in JSON format `-format=json`. Here the output is redirected to a file named `cluster-keys.json`.

Display the unseal key found in `cluster-keys.json`.

```
cat cluster-keys.json | jq -r ".unseal_keys_b64[]"

rrUtT32GztRy/pVWmcH0ZQLCCXon/TxCgi40FL1Zzus=
```

Create a variable named `VAULT_UNSEAL_KEY` to capture the Vault unseal key.

```
VAULT_UNSEAL_KEY=$(cat cluster-keys.json | jq -r ".unseal_keys_b64[]")
```

After initialization, Vault is configured to know where and how to access the storage, but does not know how to decrypt any of it. Unsealing is the process of constructing the master key necessary to read the decryption key to decrypt the data, allowing access to the Vault.

Unseal Vault running on the `vault-0` pod.

```
kubectl exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY

Key                     Value
---                     -----
Seal Type               shamir
Initialized             true
Sealed                  false
Total Shares            1
Threshold               1
Version                 1.5.4
Cluster Name            vault-cluster-4752e6ca
Cluster ID              de2b0fe9-ce24-59e5-c766-a4e2ac7df643
HA Enabled              true
HA Cluster              n/a
HA Mode                 standby
Active Node Address     <none>
Raft Committed Index    24
Raft Applied Index      24
```

The operator unseal command reports that Vault is initialized and unsealed.

Retrieve the status of Vault on the `vault-0` pod.

```
kubectl exec vault-0 -- vault status

Key                     Value
---                     -----
Seal Type               shamir
Initialized             true
Sealed                  false
Total Shares            1
Threshold               1
Version                 1.5.4
Cluster Name            vault-cluster-4752e6ca
Cluster ID              de2b0fe9-ce24-59e5-c766-a4e2ac7df643
HA Enabled              true
HA Cluster              https://vault-0.vault-internal:8201
HA Mode                 active
Raft Committed Index    29
Raft Applied Index      29
```

The Vault server is initialized and unsealed.

## Join the other Vaults to the Vault cluster

The Vault server running on the `vault-0` pod is a Vault HA cluster with a single node. To display the list of nodes requires that you are logging in with the root token.

Display the root token found in `cluster-keys.json`.
```
cat cluster-keys.json | jq -r ".root_token"
```

Create a variable named `CLUSTER_ROOT_TOKEN` to capture the Vault unseal key.

```
CLUSTER_ROOT_TOKEN=$(cat cluster-keys.json | jq -r ".root_token")
```

Login with the root token on the `vault-0` pod.
```
kubectl exec vault-0 -- vault login $CLUSTER_ROOT_TOKEN

Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                s.fgoUNVHDrdwlxftvM48A0yxa
token_accessor       vmPnI3OT0mxrI7UEa8RfJvvr
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

List all the nodes within the Vault cluster for the `vault-0` pod.
```
kubectl exec vault-0 -- vault operator raft list-peers

Node                                    Address                        State     Voter
----                                    -------                        -----     -----
09d9b35d-0336-7de7-cc94-90a1f3a0aff8    vault-0.vault-internal:8201    leader    true
```

This displays the one node within the Vault cluster. This cluster is addressable through the Kubernetes service vault-0.vault-internal created by the Helm chart. The Vault servers on the other pods need to join this cluster and be unsealed.

Join the Vault server on `vault-1` to the Vault cluster.
```
kubectl exec vault-1 -- vault operator raft join http://vault-0.vault-internal:8200

Key       Value
---       -----
Joined    true
```

This Vault server joins the cluster sealed. To unseal the Vault server requires the same unseal key, `VAULT_UNSEAL_KEY`, provided to the first Vault server.

Unseal the Vault server on `vault-1` with the unseal key.
```
kubectl exec vault-1 -- vault operator unseal $VAULT_UNSEAL_KEY

Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       1
Threshold          1
Unseal Progress    0/1
Unseal Nonce       n/a
Version            1.5.4
HA Enabled         true
```

The Vault server on `vault-1` is now a functional node within the Vault cluster.

Join the Vault server on `vault-2` to the Vault cluster using the same steps as above.


List all the nodes within the Vault cluster for the `vault-0` pod.

```
kubectl exec vault-0 -- vault operator raft list-peers

Node                                    Address                        State       Voter
----                                    -------                        -----       -----
09d9b35d-0336-7de7-cc94-90a1f3a0aff8    vault-0.vault-internal:8201    leader      true
7078a8b7-7948-c224-a97f-af64771ad999    vault-1.vault-internal:8201    follower    true
aaf46893-0a93-17ce-115e-f57033d7f41d    vault-2.vault-internal:8201    follower    true
```

This displays all three nodes within the Vault cluster.

Get all the pods within the default namespace, and confirm they are in a `READY` state.

## Set a secret in Vault

The web application that you deploy in the Launch a web application section, expects Vault to store a username and password at the path `secret/webapp/config`. To create this secret requires you to login with the root token, enable the `key-value` secret engine, and store a secret username and password at that defined path.

First, start an interactive shell session on the vault-0 pod.
```
kubectl exec --stdin=true --tty=true vault-0 -- /bin/sh
```

Enable `kv-v2` secrets at the path secret.
```
vault secrets enable -path=secret kv-v2

Success! Enabled the kv-v2 secrets engine at: secret/
```

Create a secret at path `secret/devwebapp/config` with a username and password.

```
vault kv put secret/devwebapp/config username='giraffe' password='salsa'

Key              Value
---              -----
created_time     2020-12-11T19:14:05.170436863Z
deletion_time    n/a
destroyed        false
version          1
```


Verify that the secret is defined at the path `secret/data/devwebapp/config`.
```
vault kv get secret/devwebapp/config

====== Metadata ======
Key              Value
---              -----
created_time     2020-12-11T19:14:05.170436863Z
deletion_time    n/a
destroyed        false
version          1

====== Data ======
Key         Value
---         -----
password    salsa
username    giraffe
```

You successfully created the secret for the web application.


Lastly, exit the `vault-0` pod.
```
exit
```


## Configure Kubernetes Authentication
The initial root token is a privileged user that can perform any operation at any path. The web application only requires the ability to read secrets defined at a single path. This application should authenticate and be granted a token with limited access.

Vault provides a Kubernetes authentication method that enables clients to authenticate with a Kubernetes Service Account Token.

First, start an interactive shell session on the `vault-0` pod.

```
kubectl exec --stdin=true --tty=true vault-0 -- /bin/sh
```

Enable the Kubernetes authentication method.

```
vault auth enable kubernetes

Success! Enabled kubernetes auth method at: kubernetes/
```

Vault accepts this service token from any client within the Kubernetes cluster. During authentication, Vault verifies that the service account token is valid by querying a configured Kubernetes endpoint.

Configure the Kubernetes authentication method to use the service account token, the location of the Kubernetes host, and its certificate.

```
vault write auth/kubernetes/config \
        token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
        kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
        kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt


```
The `token_reviewer_jwt` and `kubernetes_ca_cert` are mounted to the container by Kubernetes when it is created. The environment variable `KUBERNETES_PORT_443_TCP_ADDR` is defined and references the internal network address of the Kubernetes host.

For a client of the Vault server to read the secret data defined in the Set a secret in Vault step requires that the read capability be granted for the path `secret/data/devwebapp/config`.

Write out the policy named `devwebapp` that enables the read capability for secrets at path `secret/data/devwebapp/config`

```
vault policy write devwebapp - <<EOF
path "secret/data/devwebapp/config" {
  capabilities = ["read"]
}
EOF
```

Create a Kubernetes authentication role named `devweb-app`.

```
vault write auth/kubernetes/role/devweb-app \
        bound_service_account_names=internal-app \
        bound_service_account_namespaces=default \
        policies=devwebapp \
        ttl=24h

```

The role connects a Kubernetes service account, internal-app (created in the next step), and namespace, default, with the Vault policy, `devwebapp`. The tokens returned after authentication are valid for 24 hours.

Lastly, exit the vault-0 pod.

## Deploy web application

The web application pod requires the creation of the internal-app Kubernetes service account specified in the Vault Kubernetes authentication role created in the Configure Kubernetes authentication step.

Create a Kubernetes service account named `internal-app`.
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: internal-app

EOF
```

Create a pod with the web application.
```
cat <<EOF | kubectl apply -f -
---
apiVersion: v1
kind: Pod
metadata:
  name: devwebapp
  labels:
    app: devwebapp
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "devweb-app"
    vault.hashicorp.com/agent-inject-secret-credentials.txt: "secret/data/devwebapp/config"
spec:
  serviceAccountName: internal-app
  containers:
    - name: devwebapp
      image: jweissig/app:0.0.1

EOF
```

This definition creates a pod with the specified container running with the internal-app Kubernetes service account. The container within the pod is unaware of the Vault cluster. The Vault Injector service reads the annotations to find the secret path, stored within Vault at `secret/data/devwebapp/config` and the file location, `/vault/secrets/secret-credentials.txt`, to mount that secret with the pod.

Confirm `devwebapp` pod is running 

Display the secrets written to the file `/vault/secrets/secret-credentials.txt` on the `devwebapp` pod.

```
kubectl exec --stdin=true --tty=true devwebapp -c devwebapp -- cat /vault/secrets/credentials.txt

data: map[password:salsa username:giraffe]
metadata: map[created_time:2020-12-11T19:14:05.170436863Z deletion_time: destroyed:false version:1]
```

# Congrats
