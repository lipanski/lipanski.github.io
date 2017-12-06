## Notes on installing Vault

```sh
wget https://releases.hashicorp.com/vault/0.8.1/vault_0.8.1_linux_amd64.zip
unzip vault_0.8.1_linux_amd64.zip
sudo mv vault /usr/local/bin/
vault server -dev
```

Make sure to copy the unseal key and root token somewhere.

In a different session:

```sh
export VAULT_ADDR='http://127.0.0.1:8200'
vault status
vaule write secret/hello value=world
vault read secret/hello
vault read -format=json secret/hello
vault delete secret/hello
```

```sh
vault mount -path=testing generic
vault mount -path=production generic
vault mounts
vault unmount production/
```

---


```hcl
# test.hcl

storage "file" {
  path = "/home/florin/vault"
}

listener "tcp" {
  address = "127.0.0.1:8200"
  tls_disable = "true"
}
```

```sh
sudo vault server -config=test.hcl
```

In a different session:

```sh
export VAULT_ADDR='http://127.0.0.1:8200'
vault init
vault unseal
vault unseal
vault unseal
vault auth
```

```sh
vault auth-enable github
vault write auth/github/config organization=xxx
```

```hcl
# reader.hcl

path "test/*" {
  capabilities = ["read"]
}

path "auth/token/lookup-self" {
  capabilities = ["read"]
}

```

```sh
vault policy-write reader reader.hcl
vault policies
vault policies reader
vault write auth/github/map/teams/reader value=reader
```

Go to https://github.com/settings/tokens and generate a token with the `read:org` permission.

```sh
vault auth -method=github token=xxx
```
