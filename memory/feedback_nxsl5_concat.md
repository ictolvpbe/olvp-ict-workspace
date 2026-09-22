---
name: feedback-nxsl5-concat
description: "NXSL 5 (NetXMS 5/6): strings samenvoegen is '..', niet '.'. De console-Script-Library is streng, nxscript -c slikt de oude vorm nog -> een script kan lokaal compileren en toch 'syntax error' geven bij opslaan."
metadata:
  type: feedback
---

In **NXSL 5** (NetXMS 5.x/6.x, hier 6.2.3) is `.` **attribuuttoegang** geworden;
strings samenvoegen doe je met `..`. Een regel als

```
regel = node->name . " " . iface->name;     /* FOUT in v5 */
regel = node->name .. " " .. iface->name;   /* correct */
```

geeft in de Script Library van de console `Script compilation failed
(Error in line N: syntax error)`.

**Why:** de parser leest `.` achter een object als "geef attribuut" en
struikelt dan over de string die volgt. Verraderlijk detail: `"peer=" . iface->name`
geeft **geen** fout, want een string-literal heeft geen attributen — dus de
fout duikt alleen op bij de regels waar `.` achter een object of variabele
staat, en niet noodzakelijk bij de eerste concatenatie in het bestand.

**Tweede valkuil:** `nxscript -c` (de losse compiler in de netxms-server-container)
compileert in compatibiliteitsmodus en accepteert de oude `.`-vorm **wel**.
Een script kan dus lokaal schoon compileren en toch geweigerd worden bij het
opslaan in de console. `..` werkt in beide, dus schrijf altijd `..`.

**How to apply:**
- Snelle toets vóór het plakken (vangt álles behalve deze fout):
  `ssh ansible@10.35.0.20 "sudo podman exec -i netxms-server nxscript -c /dev/stdin" < script.nxsl`
- Grep op de oude vorm: `grep -n ' \. ' script.nxsl`
- Ook deprecated in 6.2 — v5 verschuift van losse functies naar methodes op het
  object: `SetInterfaceExpectedState(i,s)` → `i->setExpectedState(s)`,
  `upper(s)` → `s->toUpperCase()`, `lower(s)` → `s->toLowerCase()`.
  De console meldt die als **warning** (blokkeert opslaan noch uitvoeren);
  `nxscript -c` toont ze helemaal niet. Mogelijk volgen nog
  `GetNodeInterfaces(n)` → `n->interfaces` en `GetCustomAttribute(o,a)` →
  `o->getCustomAttribute(a)`; 6.2.3 waarschuwt daar (nog) niet over.

Gerelateerd: [[project-netxms-monitoring]] (script olvp-port-class.nxsl,
runbook RB-2026-NETXMS-PORTCLASS).
