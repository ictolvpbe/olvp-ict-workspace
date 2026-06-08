# Claude — Agent context voor OLVP Infrastructure

Geladen bij sessies in `/home/demm/SchoolProjects/olvp-ict/`. Geeft directe project-context zodat de agent productief is zonder eerst alles te moeten ontdekken.

## Wat is dit project

OLVP (Onze-Lieve-Vrouwepresentatie school) **infrastructuur-werf**. Gateway + DMZ + reverse-proxy + interne PKI + Ansible-automation + Semaphore-pipeline + multi-VM Odoo-hosting. Twee gelinkte repo's:

- **`platform-handbook/`** — docs, ADRs, runbooks (deze repo: documentatie-eerst)
- **`platform-ansible/`** — Ansible-playbooks, templates, roles voor deploy

## Werf-scope

Niet alles wat OLVP doet — alleen infra-laag. Voor app-/Odoo-werk: andere project-directory (`/home/demm/PyCharm/odoo-myschool/`). Voor de Vue/Next-frontend: weer een andere (`MySchoolAssistant`).

## Structuur

```
olvp-ict/
├── platform-handbook/        — markdown docs
│   ├── governance/           — programma-structuur, risk-register, SLA, DPIA, ITIL
│   ├── hosting/              — HAProxy, Caddy, certs, gateway
│   ├── network-physical/     — VLANs, switches
│   ├── containers/           — Podman ADRs
│   ├── management-tools/     — semaphore.md, ansible.md, bacula.md, podman.md, ...
│   └── shared/               — naming-conventies, vendors
├── platform-ansible/         — Ansible-playbooks
│   ├── site.yml, caddy.yml, addons-update.yml, forgejo.yml, odoo-podman.yml, tier1-baseline.yml
│   ├── inventory.yml         — alle VMs in haproxy/webapps/mgmt_servers groups
│   ├── vars/instances.yml    — per-Odoo-instance: env, addons_branch, fqdn, port
│   ├── group_vars/all/       — vars.yml + vault.yml (encrypted)
│   ├── templates/            — Jinja2 voor Caddyfile, app.ini, Quadlet-files
│   └── roles/odoo-podman/    — Tier 2 overlay-role voor Odoo-multi-instance
└── memory/                   — auto-memory (Claude per-project memory)
```

## Conventies

### Git
- Branch: `main` op beide repo's (geen Dev/master conventie hier — die is voor MySchool_addons)
- Commits in Nederlands, imperatief ("voeg X toe")
- Eerste regel ≤ 72 chars + optioneel body met "why"
- Co-author line `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>` toevoegen bij Claude-commits

### Ansible
- `ansible-user` op alle stack-VMs met NOPASSWD sudo (zie `hosting/operations/provision-ansible-account.md`)
- Vault-pw: `--ask-vault-pass` (copy-paste uit KeePassXC `olvp-ansible-vault-master`)
- `group_vars/all/{vars,vault}.yml` directory-pattern (zie [[project-ansible-groupvars-layout]] in memory)

### Containers
- **Podman + Quadlet** als default runtime (ADR 0004), niet Docker
- Image-pinning altijd FQDN: `docker.io/library/...`
- Quadlet-files in `/etc/containers/systemd/` op de VMs (niet `systemctl enable`, ze zijn transient)

### Netwerk + auth
- VLANs: 10 (users), 21 (DMZ-RPROXY), 22 (stepca), 34 (admin), 35 (mgmt), 36 (prod-webapps), 200/207 (test-webapps)
- step-ca op `10.21.1.10:8443`, intern PKI
- LE certs op HAProxy (publiek), step-ca certs op Caddy (re-encrypt)
- AD-DC: `SRVV-INFRA002` (LDAP/auth-bron)

### Naming
- VMs: `SRVV-<ROL>-<NN>` (zie [[project-naming-convention]])
- FQDN's: zie `platform-handbook/hosting/reference/vm-inventory.md`

## Niet doen

- ❌ Geen rechtstreekse commits op prod-config zonder branch + review-flow
- ❌ Geen destructive previews (rm/shred/luksFormat) — ALLEEN bij uitvoering
- ❌ Geen heredoc + lange `echo`-regels in runbooks — gebruik `sudo nano` voor multi-line
- ❌ Geen acties op productie-VMs (SRVV-ODOO-01, SRVV-ACC-01) zonder expliciete confirmation
- ❌ Geen `--no-verify` / `--no-gpg-sign` op git
- ❌ Geen rotaties van DB-pw of certs zonder backup en explicit user-OK

## Hot-paths

- **Memory** — altijd `memory/MEMORY.md` eerst lezen voor laatste status
- **Hosting-status** — `memory/project_hosting_fase1_status.md` = live deploy-state
- **Tasks** — context-tracker voor open werven
- **Semaphore-deploy** — Survey-vars `target_env`/`addons_branch_override`/`target_limit`, zie `platform-handbook/management-tools/semaphore.md`
- **Forgejo gepauzeerd** — `memory/project_forgejo_status.md`, postgres-pq-bug niet getraceerd

## Gerelateerde werven (andere cwds)

- **Odoo addons-dev**: `/home/demm/PyCharm/odoo-myschool/extra-addons/` (heeft eigen CLAUDE.md + memory)
- **MySchool Assistant** (Next.js + AI-service): `/home/demm/PyCharm/odoo-myschool/MySchoolAssistant/` (heeft eigen CLAUDE.md + memory)

Cross-werf raakvlakken (MCP-deploy, Odoo↔Assistant API) → cross-link in memory met `[[name]]` refs naar memory in andere project.

## Voor sub-agent gebruik

In hoofdsessie kan ik subagents inzetten voor:
- **Explore** — voor "waar staat X in deze codebase?"
- **Plan** — voor stap-voor-stap design vóór implementatie
- **infra-ops** (custom, zie `~/.claude/agents/`) — focused infra-werk in subprocess
