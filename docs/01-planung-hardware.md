# 01 – Planung & Hardware

[← Zurück zur Übersicht](../README.md)

## Anforderungen

Bevor ich etwas gekauft habe, habe ich festgehalten, was der Server können muss:

| Anforderung | Warum |
|---|---|
| **Leise** | Der Server steht im Wohnbereich. |
| **Wenig Stromverbrauch** | Läuft jeden Tag viele Stunden. |
| **Genug RAM für mehrere VMs** | Foto-Cloud, DNS, KI-Agent, später ein Security-Lab. |
| **Hardware-Virtualisierung** | Ohne Hardware-Virtualisierung keine VMs. |
| **Kabel-LAN** | Ein Server gehört ans Kabel, nicht ins WLAN. |

Die gewünschten Dienste waren: eigene Foto-Cloud, DNS-Adblocker, VPN, mehrere VMs, später Smart Home und ein eigener Sprachassistent.

## Wahl der Hardware

Gewählt habe ich einen **Mini-PC** mit **32 GB RAM** und **1 TB NVMe-SSD**.

**Warum ein Mini-PC statt eines gebrauchten Servers?**
- Deutlich **leiser** und **sparsamer** als Rack-Server.
- Genug CPU-Leistung für viele VMs gleichzeitig.
- **32 GB RAM** sind bei Virtualisierung meist der Engpass, nicht die CPU. Damit passen alle geplanten Dienste gleichzeitig drauf.

### RAM-Planung

| Dienst | geplanter RAM |
|---|---|
| Proxmox selbst | ~2 GB |
| Immich (VM) | 6 GB |
| Pi-hole (Container) | 0,5 GB |
| Claude-Code-VM | 4 GB |
| Lern-VMs (bei Bedarf) | je 2–4 GB |
| Home Assistant / Sprachassistent (später) | 2–4 GB je |

Kleine Dienste laufen als **LXC-Container** statt als VM, weil sie nur einen Bruchteil des RAMs brauchen. Lern-VMs werden nur eingeschaltet, wenn ich sie brauche.

## Boot-Stick

Für die Installation habe ich einen 64-GB-USB-Stick mit **USB-C und USB-A** gewählt. Er passt an jedes Gerät ohne Adapter und hat USB 3.2.

## Entscheidung: Proxmox VE

Verglichen habe ich:

| Option | Bewertung |
|---|---|
| **Proxmox VE** | Kostenlos, VMs **und** Container, in Firmen verbreitet → **gewählt** |
| TrueNAS SCALE | Stark als NAS, VMs eher Nebensache |
| Unraid | Einfach, aber kostenpflichtige Lizenz |
| Ubuntu + Docker | Möglich, aber keine echten VMs, weniger Lernwert |

Proxmox gibt mir saubere Trennung zwischen den Diensten und ist gleichzeitig eine Fähigkeit, die im Beruf gefragt ist.

## Vorinstalliertes Windows

Auf dem Mini-PC war Windows 11 vorinstalliert. Vor dem Überschreiben habe ich den im BIOS hinterlegten Lizenzschlüssel ausgelesen und sicher abgelegt, damit Windows bei Bedarf neu installiert werden kann:

```powershell
(Get-CimInstance -Query 'select * from SoftwareLicensingService').OA3xOriginalProductKey
```

> Der Schlüssel selbst gehört natürlich nicht in ein Repo.

---

[Weiter: 02 – Netzwerk →](02-netzwerk.md)
