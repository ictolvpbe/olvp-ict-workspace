---
name: feedback-proxmox-clone-identiteit
description: "Een Proxmox-kloon of -template vernieuwt de hardware-identiteit (VMID, MAC, smbios-uuid, vmgenid) maar kopieert de schijf byte voor byte — machine-id, SSH-host-sleutels en hostname gaan mee. qm template verandert daar niets aan. Leegmaken en afsluiten horen één handeling te zijn."
metadata:
  type: feedback
---

**Proxmox vernieuwt bij een kloon alleen de hardware-identiteit:** VMID, naam, MAC-adressen,
`smbios1: uuid` en `vmgenid`. De schijf wordt byte voor byte gekopieerd, dus álles in het
gastbestandssysteem gaat mee: `/etc/machine-id`, de SSH-host-sleutels, de hostname, de logs, de
shell-historie.

`qm template` verandert daar niets aan — het markeert de VM alleen als template en zet de schijven
read-only. Zo is [[project-template-fleet-defects]] ontstaan: twaalf VM's met dezelfde machine-id.

**Why:** systemd genereert alleen een nieuwe machine-id als `/etc/machine-id` **leeg** is (0 bytes).
Staat er een waarde in, dan blijft die er eeuwig. Een kloon "reset" dus niets; het image moet al
leeg zijn opgeborgen.

**How to apply:**

- Leegmaken en afsluiten zijn **één handeling**, zonder gaatje ertussen. Eén herstart "om iets te
  checken" en systemd vult alles weer in. Vandaar `olvp-seal-template.sh`, dat allebei doet.
- Wil je een script alleen op het template laten draaien, gebruik dan de **SMBIOS-UUID** als
  grendel — die verschilt per kloon. Een markeringsbestand is waardeloos: dat wordt juist
  meegekopieerd.
- Neem de **SSH-host-sleutels** in dezelfde beweging mee, en zet er een systemd-oneshot bij
  (`ssh-keygen -A` met `ConditionPathExistsGlob=!/etc/ssh/ssh_host_*_key`) zodat een kloon ze zelf
  aanmaakt. Debian doet dat niet uit zichzelf bij elke boot.
- Een **Proxmox-template-object kun je niet starten**. Converteer je met een gevulde machine-id,
  dan is het niet meer te herstellen zonder het template opnieuw te bouwen. OLVP houdt daarom een
  gewone VM met snapshots — zie [[project-template-strategy]].
- Cloud-init lost hostname, IP en SSH-sleutels op, maar **niet** de machine-id, tenzij het image
  hem al leeg heeft (Debian-cloudimages doen dat wel).
