---
name: feedback-no-preview-destructive
description: Destructieve commando's (shred, rm -rf, luksFormat, wipefs, passwd -l, dd of=, --no-preserve-root) nooit als 'preview' / 'wat hierna komt' tonen in code-blokken — user copy-paste leidt tot uitvoering vóór de voorgaande verificatie. Vastgelegd 2026-05-28 na onherstelbaar verlies van root_ca_key.
metadata:
  type: feedback
---

**Regel.** Toon destructieve commando's **alleen** op het moment van uitvoering, niet als preview of teaser van toekomstige stappen. Voor het vooruitkijk-deel: gebruik **proza** (Nederlandse zin met expliciete volgorde-conditie) — geen executable code-blok.

**Why.** Op 2026-05-28 toonde ik tijdens de step-ca-bringup de cleanup-commando's (`sudo shred -u /etc/step-ca/secrets/root_ca_key` etc.) in een code-blok onder een "Daarna pas:"-tekstmarker, om de user zicht te geven op de volledige flow. User kopieerde-en-plakte logisch het blok en draaide het vóór de USB-backup-stap was uitgevoerd. Resultaat: `root_ca_key` onherstelbaar geshredded zonder backup; volledige `step ca init` opnieuw nodig. De tekstmarker "Daarna pas" werd in praktijk gemist omdat het code-blok visueel domineerde. **Het is ontwerp-veiliger om de inhoud niet te tonen dan om te vertrouwen op tekst-markers die de gebruiker geacht wordt op te merken.**

**How to apply.**
- Bij destructieve commando's in code-blokken: **alleen** wanneer het de huidige stap is die ik nu vraag uit te voeren. Geen preview, geen "hier komt straks", geen "voor context".
- Vooruitkijken naar destructieve stappen: in **proza**, met conditie expliciet. Bv. "Pas na hash-verificatie van de USB-backup zullen we `shred` van de server-key doen — dat commando volgt in stap 10."
- Twijfel? Liever helemaal weglaten dan tonen. Een korte, sequentiële prompt-flow ("verifieer → bevestig → ik commit volgende stap") werkt beter dan een grote preview-dump.
- Geldt voor: `shred`, `rm -rf`, `rm` op niet-trivial paden, `cryptsetup luksFormat`, `wipefs -a`, `mkfs.*`, `dd of=...`, `passwd -l`, `userdel -r`, `git reset --hard`, `git push --force`, `git branch -D`, `truncate -s 0`, `> file` (truncate-redirect), DROP TABLE/DATABASE, en alles wat hard-to-reverse is volgens [[feedback-risk-aware-changes]].
- Workflow-blok dat ik vooraf vrijgeef als hele runbook (markdown-file met alle stappen achter elkaar) is OK — daar is de file zelf het artefact en de user verwerkt 'm als referentie, niet als kopieer-vlak. Verschil zit in **conversational/chat-output vs. opgeslagen runbook-file**.

Gerelateerd: [[feedback-risk-aware-changes]] (algemene risk-aware aanpak), [[project-internal-pki-coverage]] (de specifieke context van het incident).
