---
name: feedback-semaphore-proxyjump-prod-webapps
description: Semaphore-runner (VLAN 35 MGMT-APPLIANCES) bereikt prod-webapps (VLAN 36) alleen via bastion jump-01 — host-override ssh_common_args="" forceert direct en faalt UNREACHABLE; fix = ProxyJump-extra-var in Semaphore-Environment 'OLVP Default' + firewall A-009.
metadata:
  type: feedback
---

Een Semaphore-run tegen `srvv-odoo-01` (10.36.0.40, VLAN 36 prod-webapps) faalde `UNREACHABLE` op `:22` (2026-06-05). Oorzaak: `inventory.yml` zet host-level `ansible_ssh_common_args: ""` op srvv-odoo-01/acc-01 → ProxyJump UIT → directe SSH. Dat is bedoeld voor het **werkstation** (VLAN 34 Admin → Webapps direct open), maar de **Semaphore-runner zit in VLAN 35** (zone MGMT-APPLIANCES, 10.35.0.10) en mag VLAN 36 niet direct op poort 22.

**Fix (Optie 1, design-consistent — alles via bastion-chokepoint):**
1. UniFi firewall **A-009**: `MGMT-BASTIONS` (jump-01 10.35.0.2) → `ODOO-VMS`:22, **+ "Retourverkeer Automatisch Toestaan"** (anders SYN-ACK gedropt — zelfde als step-ca 2026-05-28).
2. Semaphore-Environment **'OLVP Default'** → Extra Variables (JSON), key toevoegen (bestaande keys zoals github_pat behouden):
   `"ansible_ssh_common_args": "-o ProxyJump=ansible@10.35.0.2"`
   Dit `-e` wint van de host-override `""` → Semaphore jumpt via jump-01; werkstation blijft direct.

**Why:** één statische host-var kan niet tegelijk "direct voor werkstation" én "via bastion voor Semaphore" zijn. De Environment-extra-var lost dat per-context op zonder de werkstation-shortcut te breken.

**How to apply:** elke nieuwe prod-webapp-VM met `ssh_common_args: ""` heeft deze Environment-var + een A-009-achtige bastion→target:22 regel nodig voordat Semaphore hem kan beheren. De hop Semaphore→jump-01:22 werkt al (test-VM's jumpen er via, regel T-006). **De Semaphore-Environment heet 'OLVP Default'** (niet "Default"). Zie [[feedback-semaphore-template-repo-vault]], [[project-firewall-strategy]], [[feedback-semaphore-key-canonical]].
