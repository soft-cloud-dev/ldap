# LDAP

1. [Wait for it](#wait-for-it)
2. [Initial setup](#initial-setup)
3. [Development](#development)
4. [Production and deployment](#production-and-deployment)

This repository: [github.com/soft-cloud-dev/ldap](https://github.com/soft-cloud-dev/ldap)

Prerequisites: *slapd*, *ldap-utils*
LDAP domain: softcloud.dev
Schemas: *cosine*, *nis*, *inetorgperson*

## Wait for it
```bash
while true ldapsearch -Y EXTERNAL -H ldapi:/// -b "cn=config" -s base >/dev/null 2>&1 -Y EXTERNAL -H ldapi:/// -b "cn=config" -s base >/dev/null 2>&1; do echo "Waiting for LDAP"; sleep 1; done
```

## Initial setup

1. [Load schemas](#load-schemas)
2. [Generate admin password hash](#generate-admin-password-hash)
3. [Set up domain name](#set-up-domain-name)
4. [Bootstrap database](#bootstrap-database)

### Load schemas
```bash
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/cosine.ldif 2>/dev/null || true
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/nis.ldif 2>/dev/null || true
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/inetorgperson.ldif 2>/dev/null || true
```

### Generate admin password hash
```bash
export ADMIN_PASS_HASH=$(slappasswd -s "$LDAP_ADMIN_PASSWORD")
```

```bash
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

### Set up domain name
```bash
export LDAP_DOMAIN=softcloud.dev
export LDAP_BASE_DN="dc=$(echo $LDAP_DOMAIN | sed 's/\./,dc=/g')"
```

### Bootstrap database
Use default `bootstrap.ldif` from this repository.
```bash
ldapadd -x -D "cn=admin,$LDAP_BASE_DN" -w "$LDAP_ADMIN_PASSWORD" -f <(curl -fsSL https://raw.githubusercontent.com/soft-cloud-dev/ldap/main/bootstrap.ldif)
```

## Development

Alternative docker setup for development is included in the repository.
```bash
git clone github.com/soft-cloud-dev/ldap
cd ldap
docker compose up
```

## Production and deployment

For production LDAP helm chart will be provided.

### Podman kube apply

A Podman-oriented Kubernetes manifest is available at [deploy/podman-kube/ldap.yaml](/Users/user/Projects/ldap/deploy/podman-kube/ldap.yaml). It follows the same basic shape as the OpenStack Helm LDAP chart:

- single LDAP server
- persistent storage for `/var/lib/ldap` and `/etc/ldap/slapd.d`
- service on port `389`
- bootstrap data loaded from this repository's [bootstrap.ldif](/Users/user/Projects/ldap/bootstrap.ldif)

The manifest uses `docker.io/osixia/openldap:1.5.0`, disables TLS for local Podman use, and publishes LDAP on host port `3389` to avoid privileged port binding problems on rootless Podman.

Before applying it, change the `stringData` values in [deploy/podman-kube/ldap.yaml](/Users/user/Projects/ldap/deploy/podman-kube/ldap.yaml):

```yaml
stringData:
  LDAP_ADMIN_PASSWORD: change-me-admin-password
  LDAP_CONFIG_PASSWORD: change-me-config-password
```

Apply the deployment:

```bash
podman kube apply -f deploy/podman-kube/ldap.yaml
```

Then test it from the host:

```bash
ldapsearch -x -H ldap://127.0.0.1:3389 -b dc=softcloud,dc=dev -D "cn=admin,dc=softcloud,dc=dev" -w "$LDAP_ADMIN_PASSWORD"
```
