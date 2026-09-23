---
name: feedback-mail-log-not-as-body
description: Rapportmail met het volledige log als body loopt stil vast op message_size_limit; stuur een samenvatting met het log gzipped als bijlage.
metadata: 
  node_type: memory
  type: feedback
  originSessionId: e193db2f-faa5-4a94-8c6d-ec2f5ad8bcf9
  modified: 2026-09-22T06:47:44.082Z
---

Een backupscript dat zijn logbestand **als body én als bijlage** meestuurt (`mail -s … -A "$LOG" … < "$LOG"`) loopt vast zodra het log groeit: postfix weigert met `fatal: message file too big` op `message_size_limit` (standaard 10 MB), en base64 maakt een bijlage nog eens ~33 % groter. Het script logt "kon niet verzenden" en gaat vrolijk door — de backup slaagt, de melding vertrekt nooit.

Gevonden op 2026-09-22 op `PC-MONITORING-01`: de USB-keten draaide correct, maar sinds **31 augustus** kwam er geen enkele rapportmail meer aan omdat het Teamdrive-log 10,2 MB was. Dezelfde constructie zat in de rol `backup-offline` (commit op main 2026-09-22) en zou dus meeverhuizen naar PC-BACKUP-01.

**Why:** dit is dezelfde klasse als de negentien fouten uit [[project-teamdrive-backup-outage-202609]] — de taak draait, maar het signaal komt niet aan. Een keten die "draait" is niet hetzelfde als een keten die meldt.

**How to apply:** body = samenvatting + laatste 40 regels + foutregels; het volledige log gzipped als bijlage zolang het onder ~4 MB blijft, anders alleen het pad vermelden. En bij elke mailende bewaking: controleer in `/var/log/mail.log` dat er een `status=sent` staat, niet alleen dat het script "verzonden" logde. Zie [[project-offline-backup-chain]].
