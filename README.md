# LDAP

OpenLDAP for **softcloud.dev** — single-server setup with bootstrap users and a Vault bind account.

## Quick start

```bash
podman kube play deploy/podman-kube/ldap.yaml
```

Re-deploy:
```bash
podman kube play --replace deploy/podman-kube/ldap.yaml
```

Tear down (keeps volumes):
```bash
podman kube down deploy/podman-kube/ldap.yaml
```

## Test

```bash
ldapsearch -x -H ldap://127.0.0.1:3389 \
  -b dc=softcloud,dc=dev \
  -D "cn=admin,dc=softcloud,dc=dev" \
  -w "$LDAP_ADMIN_PASSWORD"
```

## What's inside

| Resource | Purpose |
|---|---|
| `Secret/ldap-auth` | Admin & config passwords — **change before use** |
| `ConfigMap/ldap-bootstrap` | OUs, sample users (alice, user), Vault service account |
| `PVC/ldap-data` | Persistent LDAP database (`/var/lib/ldap`) |
| `Deployment/ldap` | `osixia/openldap:1.5.0`, TLS off, host port 3389 |
| `Service/ldap` | ClusterIP on port 389 |

## Vault integration

Bind DN: `uid=vault,ou=people,dc=softcloud,dc=dev`
Bind password: set in the bootstrap ConfigMap (`change-me-vault-password`)

On an existing instance, apply ACLs from [vault-access.ldif](vault-access.ldif):

```bash
export LDAP_BASE_DN="dc=softcloud,dc=dev"

sed "s|{{ LDAP_BASE_DN }}|$LDAP_BASE_DN|g" vault-access.ldif | \
  podman exec -i ldap-pod-ldap ldapmodify -Y EXTERNAL -H ldapi:///
```

> **Warning:** `vault-access.ldif` replaces the full `olcAccess` set on `olcDatabase={1}mdb`. Export and merge your existing ACLs first on non-fresh instances.

## Manual setup (bare metal)

<details>
<summary>Expand for raw slapd instructions</summary>

### Wait for slapd
```bash
while ! ldapsearch -Y EXTERNAL -H ldapi:/// -b "cn=config" -s base >/dev/null 2>&1; do sleep 1; done
```

### Load schemas
```bash
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/cosine.ldif 2>/dev/null || true
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/nis.ldif 2>/dev/null || true
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/inetorgperson.ldif 2>/dev/null || true
```

### Configure domain
```bash
export LDAP_DOMAIN=softcloud.dev
export LDAP_BASE_DN="dc=$(echo $LDAP_DOMAIN | sed 's/\./,dc=/g')"
export ADMIN_PASS_HASH=$(slappasswd -s "$LDAP_ADMIN_PASSWORD")

cat <<EOF | ldapmodify -Y EXTERNAL -H ldapi:///
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: $LDAP_BASE_DN
-
replace: olcRootDN
olcRootDN: cn=admin,$LDAP_BASE_DN
-
replace: olcRootPW
olcRootPW: $ADMIN_PASS_HASH
EOF
```

</details>
