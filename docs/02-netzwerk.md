# 02 – Netzwerk

[← Zurück zur Übersicht](../README.md)

## Platzhalter

In diesem Repo stehen keine echten IP-Adressen. Stattdessen:

| Platzhalter | Bedeutung |
|---|---|
| `<LAN>/24` | Mein Heimnetz (privates /24-Netz) |
| `<ROUTER-IP>` | Router = Gateway und Standard-DNS |
| `<PVE-IP>` | Proxmox-Host |
| `<IMMICH-IP>` | Immich-VM (100) |
| `<PIHOLE-IP>` | Pi-hole-Container (101) |
| `<CLAUDE-IP>` | Claude-Code-VM (102) |

Alle Server-Adressen liegen direkt hintereinander im oberen, reservierten Bereich des Netzes (siehe unten).

## IP-Plan: feste Adressen für Server

**Problem:** Der Router vergab ursprünglich den **gesamten** Adressbereich automatisch per DHCP. Eine feste IP für den Server hätte also jederzeit mit einem Handy oder Laptop kollidieren können.

**Lösung:** Den **DHCP-Pool verkleinert**, sodass der obere Bereich frei bleibt:

| Bereich | Verwendung |
|---|---|
| `.2` – `.199` | DHCP: Handys, Laptops, Gäste |
| `.200` – `.254` | **Feste IPs für Server, VMs und Container** |

**Warum so und nicht per DHCP-Reservierung?** Ein eigener Bereich für Server ist übersichtlicher und funktioniert auch dann, wenn der Router mal getauscht wird. Jeder neue Dienst bekommt einfach die nächste freie Adresse ab `.200`.

**Freie Adresse prüfen**, bevor man sie vergibt:

```powershell
ping <neue-IP>        # "Zielhost nicht erreichbar" / Timeout = frei
arp -a                # zeigt die gerade aktiven Geräte im Netz
```

## Was passiert bei einem Routerwechsel?

Proxmox und die VMs speichern IP, Gateway und DNS fest. Bekommt ein neuer Router andere Adressen, sind die Server nicht mehr erreichbar.

- **Einfachste Lösung:** Im neuen Router dasselbe Netz und dieselbe Router-IP einstellen.
- **Sonst:** Am Server per Monitor und Tastatur in `/etc/network/interfaces` und `/etc/hosts` die Adressen anpassen.

## CGNAT

Beim Prüfen des Routers kam heraus, dass meine **öffentliche IPv4-Adresse gar nicht öffentlich ist**, sondern aus einem privaten Bereich stammt. Mein Anbieter nutzt **CGNAT** (Carrier-Grade NAT): Mehrere Kunden teilen sich eine öffentliche IPv4.

**Folgen:**
- **Port-Weiterleitungen über IPv4 funktionieren nicht.** Von aussen kommt niemand direkt an meine Geräte.
- Für Fernzugriff braucht es eine Lösung, die **von innen nach aussen** verbindet → **[Tailscale](06-tailscale.md)**.

Aus Sicherheitssicht ist das eher ein Vorteil: Es gibt keine offenen Ports, die Bots finden könnten.

## DNS

- Standard-DNS im Netz ist der Router.
- Der Router erlaubt **keine eigene DNS-Einstellung** für die Geräte im Netz. Deshalb wird Pi-hole **pro Gerät** eingetragen statt netzweit (Details in [Kapitel 05](05-pihole.md)).
- Server mit fester IP bekommen den DNS **nicht automatisch** mit und brauchen einen manuellen Eintrag (siehe [Immich](04-immich.md#dns-problem-nach-der-umstellung)).

## Ports und IPv4 vs. IPv6

Eine **IP** ist die Adresse eines Hauses, ein **Port** die Wohnungsnummer. Auf einem Server laufen viele Dienste, jeder auf seinem eigenen Port:

| Port | Dienst |
|---|---|
| 22 | SSH |
| 53 | DNS (Pi-hole) |
| 80 / 443 | HTTP / HTTPS (Pi-hole-Weboberfläche) |
| 2283 | Immich |
| 8006 | Proxmox-Weboberfläche |

Offene Ports auf einem Host anzeigen:

```bash
sudo ss -tlnp          # t=TCP, l=lauschend, n=Zahlen, p=Programm
```

Der Router verteilt neben IPv4 auch **IPv6**. Das ist wichtig für die [Firewall](08-firewall.md), weil IPv6 eine zweite Route zu den Geräten öffnet.

---

[← 01 – Planung & Hardware](01-planung-hardware.md) · [Weiter: 03 – Proxmox VE →](03-proxmox.md)
