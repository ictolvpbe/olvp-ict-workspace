---
name: feedback-toon-het-echte-commando
description: "Bij een falend shell-commando uit Ansible: kijk eerst wat er FEITELIJK wordt uitgevoerd (yaml.safe_load + shlex.split) vóór je hypothesen bedenkt. Vier mislukte runs kostten dat, met drie zinloze fixes onderweg."
metadata:
  type: feedback
---

Bij het eerste e2e-draaien van `roles/frappe-podman` (2026-09-21) faalde
`bench new-site` vier runs achter elkaar. Ik bedacht telkens een nieuwe oorzaak
en voerde die meteen als fix door: shell-quoting, dan omgevingsvariabelen, dan
Podman-secrets. **Alle drie zinloos** — de echte oorzaak was een YAML folded
scalar die nieuwe regels behield, waardoor bash het commando in stukken las en
`bench` zonder enige optie werd aangeroepen.

**Why:** een fix op een onbewezen hypothese kost een volledige ronde (bij mij:
uitvoeren door de user, opnieuw diagnosticeren) en vertroebelt het beeld, want
elke wijziging voegt een variabele toe. Eén inspectie van het feitelijke
commando had het in één keer laten zien.

**How to apply:**

- Faalt een `command`/`shell`-taak: render de `cmd` en haal hem door
  `shlex.split`, en kijk naar de argv die eruit komt. Tel de `\n` in het
  bash-argument — dat hoort nul te zijn.
- `no_log: true` verbergt precies de melding die je nodig hebt. Zodra de
  wachtwoorden niet meer in de `cmd` staan, hoort hij weg.
- Draait het commando in een container met `--rm`? Laat de uitvoer dan naar een
  bestand op een bind-mount schrijven, anders is het spoor weg.
- Werkt hetzelfde commando met de hand wél en via Ansible niet, dan zit het
  verschil in hóé Ansible het aanroept — niet in de omgeving. Vergelijk de argv,
  niet de symptomen.

Gerelateerd: [[project-melira-frappe-hosting]], [[feedback-ansible-check-mode-verify]]
(zelfde grondhouding: meet op de target, vertrouw de samenvatting niet).
