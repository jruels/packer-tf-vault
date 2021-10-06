# Vault Policies

## Overview
Vault use policies to govern the behavior of clients and instrument Role-Based Access Control (RBAC) by specifying access privileges (authorization).

Vault creates a root policy during initialization. The root policy is capable of performing every operation for all paths. This policy is assigned to the root token that displays when initialization completes. This provides an initial superuser to enable secrets engines, define policies, and configure authentication methods.

Vault also creates a default, policy. The default policy defines a common set of capabilities that enable a token the ability to reflect and manage itself. This policy is also assigned to the root token.

A policy defines a list of paths. Each path expresses the capabilities that are allowed. Capabilities for a path must be granted, as Vault defaults to denying capabilities to paths to ensure that it is secure by default.

## Challenge
Since Vault centrally secures, stores, and controls access to secrets across distributed infrastructure and applications, it is critical to control permissions before any user or machine can gain access.

## Solution

Restrict the use of root policy, and write fine-grained policies to practice **least privileged**. For example, if an app gets AWS credentials from Vault, write policy grants to `read` from AWS secrets engine but not to `delete`, etc.

Policies are attached to tokens and roles to enforce client permissions on Vault.

## Write a policy


The first step is to gather policy requirements.

An admin user must be able to:

* Read system health check
* Create and manage ACL policies broadly across Vault
* Enable and manage authentication methods broadly across Vault
* Manage the Key-Value secrets engine enabled at `secret/` path

Define the admin policy in the file named `admin-policy.hcl`.
```
tee admin-policy.hcl <<EOF
# Read system health check
path "sys/health"
{
  capabilities = ["read", "sudo"]
}

# Create and manage ACL policies broadly across Vault

# List existing policies
path "sys/policies/acl"
{
  capabilities = ["list"]
}

# Create and manage ACL policies
path "sys/policies/acl/*"
{
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}

# Enable and manage authentication methods broadly across Vault

# Manage auth methods broadly across Vault
path "auth/*"
{
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}

# Create, update, and delete auth methods
path "sys/auth/*"
{
  capabilities = ["create", "update", "delete", "sudo"]
}

# List auth methods
path "sys/auth"
{
  capabilities = ["read"]
}

# Enable and manage the key/value secrets engine at `secret/` path

# List, create, update, and delete key/value secrets
path "secret/*"
{
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}

# Manage secrets engines
path "sys/mounts/*"
{
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}

# List existing secrets engines.
path "sys/mounts"
{
  capabilities = ["read"]
}
EOF

```
A policy define one or more paths and a list of permitted capabilities. Most of these capabilities map to the HTTP verbs supported by the Vault API.

The `sudo` capability allows access to paths that are root-protected (Refer to the Root protected endpoints section).

The `deny` capability disables access to the path. When combined with other capabilities it always precedence.

## Create a policy

Copy the `admin-policy.hcl` into `vault-0`
```
kubectl cp admin-policy.hcl vault-0:/home/vault/admin-policy.hcl
```

Create an *admin* policy

Create a policy named `admin` with the policy defined in `admin-policy.hcl`.

```
kubectl exec vault-0 -- vault policy write admin /home/vault/admin-policy.hcl
```
The policy is created or updated; if it already exists.

## Display a policy
List all the policies.
```
kubectl exec vault-0 -- vault policy list
```

Read the `admin` policy
```
kubectl exec vault-0 -- vault policy read admin
```

The output displays paths and capabilites defined for this policy 

## Check token capabilities
A token is able to display its capabilities for a path. This provides a way to verify the capabilities granted or denying by all of its attached policies.

Create a token with the `admin` policy attached and store the token in the variable `ADMIN_TOKEN`.
```
ADMIN_TOKEN=$(kubectl exec vault-0 -- vault token create -format=json -policy="admin" | jq -r ".auth.client_token")
```

Display the `ADMIN_TOKEN`.
```
echo $ADMIN_TOKEN

s.MdNlboI0nff3Xpo97d1TfIxd
```

The *admin* policy defines capabilities for the path `sys/auth/*`.

Retrieve the capabilities of this token for the `sys/auth/approle` path.
```
kubectl exec vault-0 -- vault token capabilities $ADMIN_TOKEN sys/auth/approle

create, delete, read, sudo, update
```

The output displays that this token has create, delete, read, sudo, update capabilities for this path.

Retrieve the capabilities of this token for a path `not` defined in the policy.

```
kubectl exec vault-0 -- vault token capabilities $ADMIN_TOKEN identity/entity

deny
```
The output displays that this token has no capabilities (`deny`) for this path.



# Congrats
