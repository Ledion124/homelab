# 08 – Firewall & Netzwerksegmentierung

[← Zurück zur Übersicht](../README.md)

## Problem

Die Claude-VM steht ganz normal im Heimnetz. Sie könnte also **jedes Gerät erreichen**: die Weboberflächen von Proxmox, Immich, Pi-hole und dem Router, aber auch Handys und Laptops der Familie. Ohne Passwörter kommt sie zwar nirgends rein, sie könnte das Netz aber **scannen** oder Dienste ohne Login ansprechen.

Die Regel in `CLAUDE.md` ist nur eine Bitte. Die Grenze muss **technisch erzwungen** werden. Das nennt man **Netzwerksegmentierung**.

## Ziel

| Verbindung | erlaubt? |
|---|---|
| VM → Internet (Anthropic API, Updates) | ✅ |
| VM → Router, DNS (Port 53) | ✅ |
| VM → alle anderen Geräte im Heimnetz | ❌ |
| Heimnetz → VM per SSH (Port 22) | ✅ |
| alles andere eingehend | ❌ |

## Schritt A: Datacenter-Firewall, ohne sich auszusperren

Die Firewall einer VM greift nur, wenn sie auch auf **Datacenter-Ebene** aktiv ist. Standardmässig würde Proxmox dann aber auch den **Host** filtern, und man kann sich aus der Weboberfläche aussperren.

*Datacenter → Firewall → Options*:
1. **Input Policy → ACCEPT** (zuerst!)
2. **Firewall → Yes**

Der Host bleibt so erreichbar wie vorher. VMs und Container ohne eigene aktive Firewall sind nicht betroffen.

## Schritt B: Regeln der VM

*102 → Firewall → Add*. Regeln werden **von oben nach unten** geprüft, die erste passende gilt. Neue Regeln landen in Proxmox **oben**. Darum das Verbot **zuerst** anlegen, dann die Ausnahmen.

| # | Richtung | Aktion | Quelle | Ziel | Protokoll | Port | Zweck |
|---|---|---|---|---|---|---|---|
| 0 | out | ACCEPT | | `<ROUTER-IP>` | udp | 53 | DNS |
| 1 | out | ACCEPT | | `<ROUTER-IP>` | tcp | 53 | DNS |
| 2 | out | **DROP** | | `<LAN>/24` | | | Heimnetz sperren |
| 3 | in | ACCEPT | `<LAN>/24` | | tcp | 22 | SSH von zuhause |

**Warum funktioniert Internet trotz Regel 2?** Pakete ins Internet haben als Ziel eine öffentliche IP, keine Adresse im Heimnetz. Regel 2 trifft nicht zu. Der Router ist nur der Weg dorthin.

> **Lektion:** Beim Einfügen von `x.x.x.x/24` meldete Proxmox *„not a valid IP or CIDR"*. Beim Kopieren war ein **unsichtbares Zeichen** mitgekommen. IP-Bereiche lieber von Hand tippen.

## Schritt C: Firewall der VM einschalten

*102 → Firewall → Options*:
- **Input Policy: DROP** (alles Eingehende blocken, ausser Regel 3)
- **Output Policy: ACCEPT** (alles Ausgehende erlauben, ausser Regel 2)
- **Firewall: Yes**

Bestehende SSH-Verbindungen bleiben dank Connection Tracking offen.

## Schritt D: IPv6 in der VM abschalten

Die Regeln gelten für **IPv4**. Über **IPv6** haben die anderen Geräte aber eine zweite Adresse, eine Hintertür. Claude Code braucht kein IPv6:

```bash
echo 'net.ipv6.conf.all.disable_ipv6 = 1' | sudo tee /etc/sysctl.d/99-no-ipv6.conf
echo 'net.ipv6.conf.default.disable_ipv6 = 1' | sudo tee -a /etc/sysctl.d/99-no-ipv6.conf
sudo sysctl -p /etc/sysctl.d/99-no-ipv6.conf
ip a | grep inet6      # keine Ausgabe = IPv6 aus
```

## Schritt E: Testen

Eine Sicherheitsregel ist erst etwas wert, wenn sie getestet ist, und zwar in **beide** Richtungen: Blockiert sie, was sie soll, und lässt sie durch, was nötig ist?

```bash
ping -c 2 deb.debian.org                     # ✅ antwortet (Internet + DNS)
ping -c 2 -W 2 <IMMICH-IP>                   # ✅ 100 % packet loss
curl -m 5 -k https://<PVE-IP>:8006           # ✅ Timeout
```

Tatsächliches Ergebnis:

```text
2 packets transmitted, 2 received, 0% packet loss       ← Internet
2 packets transmitted, 0 received, 100% packet loss     ← Immich gesperrt
curl: (28) Connection timed out after 5004 milliseconds ← Proxmox gesperrt
```

Zusätzlich geprüft: SSH vom Laptop in die VM ✅, Claude-App am Handy antwortet ✅.

Danach: neuer **Snapshot** des sauberen Stands.

## Roadmap

- [ ] **Proxmox-Host-Firewall:** Web-UI (8006) und SSH (22) nur aus dem Heimnetz bzw. Tailnet. Vorsicht, Aussperr-Gefahr: nur mit Monitor und Tastatur am Server als Notfallzugang
- [ ] Separates, isoliertes Netz (VLAN / eigene Bridge) für ein Security-Lab

---

[← 07 – Claude-Code-VM](07-claude-code-vm.md) · [Weiter: 09 – Betrieb →](09-betrieb.md)
