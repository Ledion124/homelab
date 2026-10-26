# 03 – Proxmox VE

[← Zurück zur Übersicht](../README.md)

## 1. ISO herunterladen und prüfen

**Was:** Proxmox VE 9.2 ISO von der offiziellen Seite `proxmox.com/en/downloads` heruntergeladen.

**Warum prüfen:** Eine manipulierte ISO wäre der denkbar schlechteste Start für einen Server. Darum habe ich den **SHA256-Hash** der Datei mit dem auf der Download-Seite angegebenen Wert verglichen:

```powershell
certutil -hashfile proxmox-ve_9.2-1.iso SHA256
```

Stimmt der Hash exakt, ist die Datei unverändert und vollständig.

## 2. Boot-Stick erstellen

**Was:** Mit **Rufus** (Herausgeber geprüft: „Akeo Consulting") auf den USB-Stick geschrieben.

**Warum DD-Modus:** Proxmox-ISOs müssen **1:1** auf den Stick kopiert werden (DD-Image-Modus), sonst bootet der Installer oft nicht. Rufus erzwingt das bei dieser ISO automatisch.

> Windows meldet danach eventuell, der Stick müsse formatiert werden. **Nicht formatieren**, Windows kann das Linux-Dateisystem nur nicht lesen.

## 3. BIOS

- **Hardware-Virtualisierung** muss im BIOS aktiv sein.
- Vom USB-Stick booten (Boot-Menü).

## 4. Installation

| Einstellung | Wert | Warum |
|---|---|---|
| Ziel-Disk | 1-TB-NVMe, **ext4** | Einfach und robust für eine einzelne SSD |
| Land / Zeitzone / Tastatur | Schweiz / Europe/Zurich / de-ch | |
| root-Passwort | lange Passphrase | root kontrolliert alle VMs |
| Management Interface | LAN-Port mit Link | Server nie über WLAN |
| Hostname | `pve.home` | |
| IP | `<PVE-IP>/24` (fester Bereich) | Siehe [Netzwerk](02-netzwerk.md) |
| Gateway / DNS | `<ROUTER-IP>` | |

Der Installer füllt die Felder teilweise mit **IPv6-Adressen** vor. Diese müssen durch die IPv4-Werte ersetzt werden.

Danach ist die Weboberfläche unter `https://<PVE-IP>:8006` erreichbar.

**Warum Port 8006?** Proxmox nutzt bewusst nicht 443, damit dieser Port für andere Dienste frei bleibt. Die Zertifikatswarnung im Browser kommt vom **selbst ausgestellten** Zertifikat. Die Verbindung ist trotzdem verschlüsselt.

## 5. Update-Quellen umstellen

**Problem:** Standardmässig sind die **Enterprise-Repositories** aktiv. Die sind nur mit Abo nutzbar. Ohne Abo schlägt jedes Update mit einem Fehler fehl, und der Server bekäme **keine Sicherheitsupdates**.

**Lösung:** Unter *pve → Updates → Repositories*:
- `pve-enterprise` und `ceph … enterprise` → **Disable** (nicht löschen)
- **Add → No-Subscription**

Die No-Subscription-Quelle ist kostenlos, enthält dieselbe Software und ist für Homelabs gedacht.

## 6. Updates einspielen

*pve → Updates → Refresh*, dann *Upgrade* (bzw. in der Shell):

```bash
apt update && apt full-upgrade
reboot          # nötig, wenn ein neuer Kernel installiert wurde
```

Dabei kam unter anderem ein **Microcode-Update** gegen eine CPU-Sicherheitslücke. Die Info-Seite von `apt-listchanges` beendet man mit `q`.

## 7. Virtualisierung prüfen

```bash
grep -cE 'svm|vmx' /proc/cpuinfo   # zählt Threads mit aktiver Virtualisierung
nproc                              # Anzahl Threads zum Vergleich
lscpu | grep -i virtual            # zeigt die Virtualisierungs-Technik
```

Sind beide Zahlen gleich und grösser als 0, können alle Threads VMs ausführen.

## 8. Zwei-Faktor-Login

**Was:** *Datacenter → Permissions → Two Factor* → **TOTP** (Authenticator-App) und zusätzlich **Recovery Keys**.

**Warum:** Ein gestohlenes Passwort allein reicht dann nicht mehr. Die Recovery Keys verhindern, dass ich mich aussperre, falls das Handy verloren geht.

> **Lektion:** QR-Codes und TOTP-Secrets nie fotografieren oder teilen. Ein Foto davon reicht, um die 2FA-Codes nachzubauen. Ist das passiert: Secret mit **Randomize** neu erzeugen.

## 9. Start beim Booten

Pro VM/Container unter *Options → Start at boot* aktiviert, mit **Start/Shutdown order**:

| Reihenfolge | Dienst | Warum |
|---|---|---|
| 1 | Pi-hole (101) | DNS zuerst |
| 2 | Immich (100) | |
| 3 | Claude-VM (102) | |

---

[← 02 – Netzwerk](02-netzwerk.md) · [Weiter: 04 – Immich →](04-immich.md)
