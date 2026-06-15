---
name: feedback-semaphore-key-canonical
description: Bij Semaphore Inventory-setup voor OLVP altijd `ansible-targets-ssh` als SSH-key gebruiken — niet `ansible-service-account` (geplande naam, geeft 'parsing private key: ssh: no key found')
metadata:
  type: feedback
---

In Semaphore Key Store is **`ansible-targets-ssh`** de canonieke werkende SSH-key voor alle OLVP-target-VMs. Een eerder geplande entry `ansible-service-account` is in de UI aangemaakt maar nooit met valid key-content gevuld → falende runs met `Failed to install inventory: parsing private key: ssh: no key found`.

**Why**: Geconstateerd 2026-06-02 bij eerste run van `Odoo extra-addons Update` template. HAProxy-template gebruikte al `ansible-targets-ssh` succesvol; Addons-template wees naar de lege `ansible-service-account`-entry. Eenduidigheid op één key is bovendien gewenst zolang we geen onderscheid tussen rollen forceren (we hebben één `ansible`-user op alle stack-VMs, [[feedback-service-accounts]]).

**How to apply**:
- Bij nieuwe Inventory aanmaken in Semaphore: User Credentials = `ansible-targets-ssh`
- `ansible-service-account` in Key Store laten staan kan, maar gebruik 'm pas als we expliciet een aparte service-account-key uitrollen (geen acute reden)
- Als nieuwe doc verwijst naar `ansible-service-account`: corrigeer in dezelfde commit (management-tools/semaphore.md was reeds gecorrigeerd op 2026-06-02)

**Naam-val (2026-06-15):** de canonieke key onder Semaphore-credential `ansible-targets-ssh` heeft als **key-comment** net `ansible-service-account` (fp `SHA256:DpQLou…`). De Semaphore-*credential* `ansible-targets-ssh` en de key-*comment* `ansible-service-account` slaan dus op dezelfde key — verwar de credential-naam niet met het key-comment, en niet met de lege/kapotte gelijknamige credential. Zie [[project-workstation-ansible-key]] (waar dit een valse "verkeerde key"-diagnose veroorzaakte).

Gerelateerd: [[feedback-service-accounts]], [[project-hosting-fase1-status]] (Semaphore-runner SSH-pad).
