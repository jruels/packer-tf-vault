# Vault Secrets

## Overview

## Challenge
The KV secrets engine v1 does not provide a way to version or roll back secrets. This made it difficult to recover from unintentional data loss or overwrite when more than one user is writing at the same path.

## Solution
Run the version 2 of KV secrets engine which can retain a configurable number of secret versions. This enables older versions' data to be retrievable in case of unwanted deletion or updates of the data. In addition, its Check-and-Set operations can be used to protect the data from being overwritten unintentionally.

![key-value versioning](https://learn.hashicorp.com/img/vault-versioned-kv-1.png)
## Step 1: Enable the KV secrets engine version 2
```
kubectl exec vault-0 -- vault kv enable-versioning secret/
```

Confirm KV engine version 2 is enabled.
```
kubectl exec vault-0 -- vault secrets list -detailed

Path          Type         Accessor           ...   Options           Description
----          ----         --------                 -------           -----------
cubbyhole/    cubbyhole    cubbyhole_9d52aeac ...   map[]             per-token private secret storage
identity/     identity     identity_acea5ba9  ...   map[]             identity store
secret/       kv           kv_2226b7d3        ...   map[version:2]    key/value secret storage
...
```

## Step 2: Write secrets
To understand how the versioning works, let's write some test data.

Create a secret at the path `secret/customer/acme` with a `name` and `contact_email`.
```
kubectl exec vault-0 -- vault kv put secret/customer/acme name="ACME Inc." \
        contact_email="jsmith@acme.com"

Key              Value
---              -----
created_time     2018-04-14T00:05:47.115378933Z
deletion_time    n/a
destroyed        false
version          1
```
The secret stores metadata along with the secret data. The version is an auto-incrementing number that starts at `1`.


Create secret at the same path `secret/customer/acme` but with different secret data.
```
kubectl exec vault-0 -- vault kv put secret/customer/acme name="ACME Inc." contact_email="john.smith@acme.com"

Key              Value
---              -----
created_time     2018-04-14T00:13:35.296018431Z
deletion_time    n/a
destroyed        false
version          2
```

This secret data is replaced and the version is incremented to `2`.

Get the secret defined at the path `secret/customer/acme`.
```
kubectl exec vault-0 -- vault kv get secret/customer/acme

====== Metadata ======
Key              Value
---              -----
created_time     2018-04-14T00:13:35.296018431Z
deletion_time    n/a
destroyed        false
version          2

======== Data ========
Key              Value
---              -----
contact_email    john.smith@acme.com
name             ACME Inc.
```

The output displays the second secret. Creating a secret at the same path replaces the existing data; fields are not merged together. The secret data stored in earlier versions is still accessible but it is no longer returned by default.

Patch the secret defined at the path `secret/customer/acme` with a new `contact_email`.
```
kubectl exec vault-0 -- vault kv patch secret/customer/acme contact_email="admin@acme.com"
```

The patch commands merges the fields within the secret data.

## Step 3: Retrieve a specific version of secret
You may run into a situation where you need to view the secret before an update.

Get version 1 of the secret defined at the path `secret/customer/acme`.
```
kubectl exec vault-0 -- vault kv get -version=1 secret/customer/acme

====== Metadata ======
Key              Value
---              -----
created_time     2018-04-14T00:05:47.115378933Z
deletion_time    n/a
destroyed        false
version          1

======== Data ========
Key              Value
---              -----
contact_email    jsmith@acme.com
name             ACME Inc.
```

Get the metadata of the secret defined at the path `secret/customer/acme`.
```
kubectl exec vault-0 -- vault kv metadata get secret/customer/acme

========== Metadata ==========
Key                     Value
---                     -----
cas_required            false
created_time            2019-07-31T05:57:32.959408Z
current_version         2
delete_version_after    0s
max_versions            0
oldest_version          0
updated_time            2019-07-31T05:58:45.338017Z

====== Version 1 ======
Key              Value
---              -----
created_time     2019-07-31T05:57:32.959408Z
deletion_time    n/a
destroyed        false

====== Version 2 ======
Key              Value
---              -----
created_time     2019-07-31T05:58:06.354642Z
deletion_time    n/a
destroyed        false
```

## Step 4: Specify the number of versions to keep
By default, the `kv-v2` secrets engine keeps up to 10 versions. Let's limit the maximum number of versions to keep to be 4.

Configure the secrets engine at path `secret/` to limit all secrets to a maximum of `4` versions.
```
kubectl exec vault-0 -- vault write secret/config max_versions=4
```

Every secret stored for this engine are set to a maximum of `4` versions.

Display the secrets engine configuration settings.
```
kubectl exec vault-0 -- vault read secret/config

Key             Value
---             -----
cas_required    false
max_versions    4
```

Configure the secret at path `secret/customer/acme` to limit secrets to a maximum of `4` versions.

```
kubectl exec vault-0 -- vault kv metadata put -max-versions=4 secret/customer/acme
```

The secret can also define the maximum number of versions.

Create four more secrets at the path `secret/customer/acme`.

```
kubectl exec vault-0 -- vault kv put secret/customer/acme name="ACME Inc." \
        contact_email="admin@acme.com"
```

Get the metadata of the secret defined at the path `secret/customer/acme`.

```
kubectl exec vault-0 -- vault kv metadata get secret/customer/acme

======= Metadata =======
Key                Value
---                -----
created_time       2018-04-14T00:42:25.677078177Z
current_version    6
max_versions       0
oldest_version     3
updated_time       2018-04-16T00:17:23.930473344Z

====== Version 3 ======
Key              Value
---              -----
created_time     2018-04-16T00:15:59.880368849Z
deletion_time    n/a
destroyed        false

====== Version 4 ======
Key              Value
---              -----
created_time     2018-04-16T00:16:18.941331243Z
deletion_time    n/a
destroyed        false

====== Version 5 ======
Key              Value
---              -----
created_time     2018-04-16T00:16:34.407951572Z
deletion_time    n/a
destroyed        false

====== Version 6 ======
Key              Value
---              -----
created_time     2018-04-16T00:17:23.930473344Z
deletion_time    n/a
destroyed        false
```

The metadata displays the `current_version` and the history of versions stored. Secrets stored at this path are limited to 4 versions. Version 1 and 2 are deleted.

Verify that version 1 of the secret defined at the path `secret/customer/acme` are deleted.
```
kubectl exec vault-0 -- vault kv get -version=1 secret/customer/acme

No value found at secret/data/customer/data
```
## Step 5: Delete versions of secret
Delete version 4 and 5 of the secrets at path `secret/customer/acme`.

```
kubectl exec vault-0 -- vault kv delete -versions="4,5" secret/customer/acme
```

Get the metadata of the secret defined at the path `secret/customer/acme`.
```
vault kv metadata get secret/customer/acme

====== Version 4 ======
Key              Value
---              -----
created_time     2018-04-16T00:12:25.404198622Z
deletion_time    2018-04-16T01:04:01.160426888Z
destroyed        false

====== Version 5 ======
Key              Value
---              -----
created_time     2018-04-16T00:12:47.527981267Z
deletion_time    2018-04-16T01:04:01.160427742Z
destroyed        false
...
```

The metadata on versions 4 and 5 reports its deletion timestamp (`deletion_time`); however, the `destroyed` parameter is set to `false`.

Undelete version 5 of the secrets at path `secret/customer/acme`.
```
kubectl exec vault-0 -- vault kv undelete -versions=5 secret/customer/acme
```
## Step 6: Permanently delete data

Destroy version `4` of the secrets at path `secret/customer/acme`.
```
kubectl exec vault-0 -- vault kv destroy -versions=4 secret/customer/acme

Success! Data written to: secret/destroy/customer/acme
```

Get the metadata of the secret defined at the path `secret/customer/acme`.
```
kubectl exec vault-0 -- vault kv metadata get secret/customer/acme

# ...snip...
====== Version 4 ======
Key              Value
---              -----
created_time     2018-04-16T00:12:25.404198622Z
deletion_time    2018-04-16T01:04:01.160426888Z
destroyed        true
# ...snip...
```

The metadata displays that Version 4 is destroyed.

Delete all versions of the secret at the path `secret/customer/acme`.
```
kubectl exec vault-0 -- vault kv metadata delete secret/customer/acme

Success! Data deleted (if it existed) at: secret/metadata/customer/acme
```

## Step 7: Configure automatic data deletion

As of Vault 1.2, you can configure the length of time before a version gets deleted. For example, if your organization requires data to be deleted after 10 days from its creation, you can configure the K/V v2 secrets engine to do so by setting the `delete_version_after` parameter.

Configure the secrets at path `secret/test` to delete versions after `40` seconds.
```
kubectl exec vault-0 -- vault kv metadata put -delete-version-after=40s secret/test

Success! Data written to: secret/metadata/test
```

Create a secret at the path `secret/test`.
```
kubectl exec vault-0 -- vault kv put secret/test message="data1"
```

Again, create a secret at the path `secret/test`.
```
kubectl exec vault-0 -- vault kv put secret/test message="data2"
```

Again, create a secret at the path `secret/test`.
```
kubectl exec vault-0 -- vault kv put secret/test message="data3"
```

Get the metadata of the secret defined at the path `secret/test`.
```
kubectl exec vault-0 -- vault kv metadata get secret/test

========== Metadata ==========
Key                     Value
---                     -----
cas_required            false
created_time            2019-07-31T06:51:49.726188Z
current_version         3
delete_version_after    40s
max_versions            0
oldest_version          0
updated_time            2019-07-31T06:55:54.203923Z

====== Version 1 ======
Key              Value
---              -----
created_time     2019-07-31T06:55:37.795176Z
deletion_time    2019-07-31T06:56:17.795176Z
destroyed        false

====== Version 2 ======
Key              Value
---              -----
created_time     2019-07-31T06:55:41.207457Z
deletion_time    2019-07-31T06:56:21.207457Z
destroyed        false

====== Version 3 ======
Key              Value
---              -----
created_time     2019-07-31T06:55:43.811997Z
deletion_time    2019-07-31T06:56:23.811997Z
destroyed        false
```

The metadata displays a `deletion_time` set on each version. After `40` seconds, the data gets deleted automatically. The data has not been destroyed.

Get version 1 of the secret defined at the path `secret/test`.

```
kubectl exec vault-0 -- vault kv get -version=1 secret/test

====== Metadata ======
Key              Value
---              -----
created_time     2019-07-31T06:55:37.795176Z
deletion_time    2019-07-31T06:56:17.795176Z
destroyed        false
version          1
```

## Step 8: Check-and-Set operations
The v2 of KV secrets engine supports a Check-And-Set operation to prevent unintentional secret overwrite. When you pass the `cas` flag to Vault, it first checks if the key already exists.

Display the secrets engine configuration settings.
```
 kubectl exec vault-0 -- vault read secret/config

Key             Value
---             -----
cas_required    false
max_versions    0
```

The `cas_required` setting is `false`. The KV secrets engine defaults to disable the Check-And-Set operation.

Configure the secrets engine at path `secret/` to enable Check-And-Set.
```
kubectl exec vault-0 -- vault write secret/config cas-required=true
```

Configure the secret at path `secret/partner` to enable Check-And-Set.
```
 kubectl exec vault-0 -- vault kv metadata put -cas-required=true secret/partner
```

Once check-and-set is enabled, every write operation requires the cas parameter with the current version of the secret. Set `cas` to `0` when a secret at that path does not already exist.

Create a new secret at the path `secret/partner`.
```
kubectl exec vault-0 -- vault kv put -cas=0 secret/partner name="Example Co." partner_id="123456789"

Key              Value
---              -----
created_time     2018-04-16T22:58:15.798753323Z
deletion_time    n/a
destroyed        false
version          1
```

Overwrite the secret at the path `secret/partner`.
```
kubectl exec vault-0 -- vault kv put -cas=1 secret/partner name="Example Co." \
      partner_id="ABCDEFGHIJKLMN"
Key              Value
---              -----
created_time     2018-04-16T23:00:28.66552289Z
deletion_time    n/a
destroyed        false
version          2

Key              Value
---              -----
created_time     2018-04-16T23:00:28.66552289Z
deletion_time    n/a
destroyed        false
version          2
```

# Congrats!
