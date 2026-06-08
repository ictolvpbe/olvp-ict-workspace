---
name: feedback-keycloak-ldaps-truststore
description: Keycloak (Java/Quarkus) heeft `KC_TRUSTSTORE_PATHS` met AD-DC cert nodig voor LDAPS-federatie naar interne CA. Plus postgres:15 (Debian) uid 999 — niet 70 zoals alpine. Beide gotchas opgelost in roles/keycloak/ op 2026-06-04.
metadata:
  type: feedback
---

**Twee gotchas bij Keycloak-deploy via Podman+Quadlet** (geconstateerd 2026-06-04 PoC #43):

## 1. LDAPS-handshake faalt zonder explicit truststore

Java cacerts in de Keycloak-image kent geen interne AD-DC certs → LDAP user-federation klikt "Test connection" → **handshake failed**.

**Fix**: `KC_TRUSTSTORE_PATHS` env-var + bind-mount-folder met cert(en):

```yaml
# Quadlet template
Environment=KC_TRUSTSTORE_PATHS=/opt/keycloak/truststore
Volume=/opt/keycloak/truststore:/opt/keycloak/truststore:ro

# Ansible task fetcht cert via openssl s_client (idempotent, refresh > 30d before expiry):
- name: AD-DC cert ophalen via openssl s_client
  shell: |
    echo | openssl s_client -connect <host>:636 -servername <host> -showcerts 2>/dev/null \
      | sed -n '/BEGIN CERTIFICATE/,/END CERTIFICATE/p' > /opt/keycloak/truststore/<host>.crt
```

Keycloak combineert de extra certs met default Java cacerts → LDAPS-handshake werkt.

## 2. postgres:15 (Debian) uid = 999, NIET 70

Forgejo + ACC-01 memory suggereerden uid 70 (alpine). Voor `docker.io/library/postgres:15` (Debian-based, default) is het **uid 999**. Verkeerde uid op bind-mount → `FATAL: could not open file "global/pg_filenode.map": Permission denied` → crash-loop.

**Fix**: chown -R 999:999 op bind-mount, of switch naar `postgres:15-alpine` voor uid 70.

```yaml
- name: Postgres data-dir aanmaken + recursive chown
  ansible.builtin.file:
    path: /opt/<service>/db
    state: directory
    owner: '999'
    group: '999'
    mode: '0700'
    recurse: true   # belangrijk: ook chown van bestaande subdirs
```

Postgres-entrypoint chowned bij eerste run zelf naar postgres-uid, maar als Ansible **vóór** start een verkeerde uid heeft gechowned, crash-cycle tot postgres-entrypoint het uiteindelijk fixt (kan 5-30 retries duren).

## 3. AD MaxPageSize=1000 cap zonder Keycloak LDAP-pagination

Zonder pagination-toggle in LDAP-provider stopt sync op exact 1000 users (AD's default `MaxPageSize`). Voor OLVP (~3000 users incl. leerlingen/ouders/staff/service-accounts) cruciaal.

**Fix**: in Keycloak LDAP-provider-page toggle **`Pagination`** = **ON** + Save + re-sync. Gebruikt LDAP RFC 2696 paged-search-control (AD-compatibel out-of-the-box). PoC 2026-06-04 ging van 1000 → 3075 users na pagination-toggle.

**Verifie**: `podman exec keycloak-db psql -U keycloak -d keycloak -tAc "SELECT COUNT(*) FROM user_entity u JOIN realm r ON u.realm_id=r.id WHERE r.name='olvp' AND u.federation_link IS NOT NULL"`

## How to apply

- **Bij elke nieuwe IdP/auth-deploy**: cert-truststore-pattern overwegen voor LDAPS-naar-interne-CA
- **Bij postgres-bind-mount**: check image-variant — Debian = 999, alpine = 70
- **Bij LDAP-federatie > 1000 users**: pagination-toggle aan vóór eerste sync
- **Bij twijfel**: `podman exec <db-container> id postgres` voor actuele uid in container

## Gerelateerd

- [[project-forgejo-status]] — postgres-uid was ook issue daar (toen alpine, dus 70)
- [[feedback-odoo19-podman-uid]] — Odoo:19.0 container draait als uid 100 (analoge container-uid-bewustzijn)
- `platform-handbook/hosting/decisions/0006-dedicated-idp-keycloak.md` — ADR
- `platform-ansible/roles/keycloak/` — beide fixes geïntegreerd
