---
name: project-network-detection
description: "Netwerkdetectie (NSM) voor OLVP: Suricata-sensor i.p.v. volledige Security Onion, ADR 0007 (2026-08-30). Fase 4, met UniFi IDS/IPS detection-modus als goedkope meetstap vooraf. Risk-item R-30."
metadata:
  node_type: memory
  type: project
---

OLVP kiest voor netwerkdetectie een **kale Suricata-sensor** (`SRVV-IDS-01`, switch-port-mirror, EVE-JSON → Wazuh/Loki, visualisatie in de bestaande Grafana) en **niet** voor een volledige Security Onion-distributie. Vastgelegd in [ADR 0007](platform-handbook/hosting/decisions/0007-netwerkdetectie-suricata-sensor.md), risk-item **R-30**, ingepland in Fase 4 (zie [[project-phasing]]).

**Why:** Netwerk-gedrag is de ontbrekende vierde as naast beschikbaarheid ([[project-netxms-monitoring]]), host-integriteit (Wazuh) en perimeter ([[project-firewall-strategy]]). Een aanvaller die na app-exploit in-memory blijft en naar buiten belt, is host-zijdig onzichtbaar. Security Onion is inhoudelijk de sterkste gratis NSM-distro, maar werd afgewezen op operationele draagbaarheid: het is gebouwd rond een SOC-triage-workflow, en een IDS zonder dagelijkse triage geeft vals dekkingsgevoel — doorslaggevend bij [[project-bus-factor]] (R-04, score 16). Bijkomend: ~8 vCPU/32 GB, een tweede Elastic-pane-of-glass naast Loki/Grafana + Wazuh, beperkt payload-zicht door TLS, en DPIA-last bij leerlingenverkeer (minderjarigen).

De kale Suricata-sensor levert ~70% van de detectiewaarde (zelfde engine, zelfde ET-ruleset), hergebruikt de pane-of-glass die er toch komt, past in het Tier 1/2 Ansible-model ([[project-template-strategy]]) en blokkeert geen enkel later pad naar volledige Security Onion.

**Volgorde:**
1. **Meetstap (gratis, mag vroeger dan Fase 4):** UniFi IDS/IPS in *detection*-modus, vier schoolweken alert-volume + ruis vastleggen. UniFi IDS is onder de motorkap ook Suricata+ET, maar zonder tuning/historiek/integratie → bruikbaar als meetinstrument, niet als detectielaag.
2. **Sensor:** Ansible-role `suricata-sensor`, pas bouwen ná de meetstap. Zeek optioneel als tweede stap. Full-PCAP bewust niet.
3. **Volledige Security Onion:** pas wanneer er een tweede operationele beheerder is **én** een vastgelegd triage-ritme. Dat is de her-evaluatie-trigger van de ADR.

**Openstaand:** verifiëren welke switch de gateway-uplink draagt en of port-mirroring daar kan; DPIA-sectie "netwerkmonitoring" (retentie, toegang, expliciet géén full-PCAP van VLAN 10) vóór activatie; bij Wazuh-uitrol meteen Suricata-decoders + alert-mapping voorzien.

**How to apply:** Bij vragen over IDS/IPS/SIEM/NSM eerst naar deze beslissing verwijzen en niet vooruitlopen op de fasering. Prevention-modus (inline) nooit vóór er detectie-data is die de false-positive-ruis kwantificeert — elke FP is een productie-incident op publieke diensten.
