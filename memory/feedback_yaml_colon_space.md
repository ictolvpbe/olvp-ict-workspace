---
name: feedback-yaml-colon-space
description: Wanneer YAML-gotchas documenteren: schrijf "spatie na de :" niet "juiste indentatie". Eenduidigheid wint van technische correctheid bij gebruikersdocs.
metadata:
  type: feedback
---

Bij het documenteren van YAML-syntax-fouten: noem het probleem **expliciet en eenduidig**, niet algemeen.

❌ "Zorg voor juiste indentatie" — technisch correct maar te vaag, lezer weet niet wat te checken.
✓ "Plaats een spatie tussen `:` en de waarde" — exacte, controleerbare actie.

**Concreet voorval (2026-05-21)**: vault-bestand bevatte `vault_keepalived_auth_pass:ss1BRl...` (geen spatie na `:`). Semaphore faalde met "Invalid variable file contents" op regel 2. `ansible-vault view` toonde decrypted content zonder duidelijke fout. Pas bij laden door Ansible/Semaphore viel het door de mand.

**Why:** De gebruiker gaf expliciete feedback: *"De formulering juiste identatie is technisch wel ok maar niet eenduidig."* Bij niet-eenduidige formulering moet hij/iemand-anders later opnieuw raden wat er bedoeld wordt. Doc-leerwaarde verdampt.

**How to apply:**
- Bij iedere YAML/JSON/config-gotcha in docs: noem de exacte syntax-positie (welk teken, welke spatie, welke quote) i.p.v. abstracte termen als "syntax", "indentatie", "formatting".
- Toon **fout vs. correct** zij-aan-zij met een leesbare visuele marker (`❌` / `✓`).
- Vermeld het exacte error-bericht uit de tool zodat zoeken op die string later naar deze gotcha leidt.
- Gotcha-blokken altijd inline bij de stap waar ze relevant zijn, niet alleen in een aparte troubleshooting-sectie achteraan.

Gerelateerd: [[project-containerization]] (NoNewPrivileges + iptables-gotchas), [[feedback-zijsporen-welkom]] (proactief documenteren wat we leren).
