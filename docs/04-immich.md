# 04 – Immich (Foto-Cloud)

[← Zurück zur Übersicht](../README.md)

**Immich** ist eine selbst gehostete Alternative zu Google Fotos, mit Apps für iOS/Android, Gesichtserkennung und geteilten Alben. Ziel: Familienfotos aus OneDrive auf den eigenen Server holen.

Aufbau: **Proxmox → Debian-VM → Docker → Immich**

## 1. VM erstellen

Die Debian-13-netinst-ISO wurde direkt in Proxmox geladen (*local → ISO Images → Download from URL*), ohne USB-Stick.

| Einstellung | Wert | Warum |
|---|---|---|
| VM-ID / Name | 100 / `immich` | |
| Qemu Agent | an | IP-Anzeige in Proxmox, sauberes Herunterfahren, konsistente Backups |
| Disk | 50 GB, **Discard** an | Thin-Provisioning: belegt nur, was genutzt wird. Discard gibt gelöschten Platz an die SSD zurück |
| CPU | 4 Kerne, Typ `host` | Echte CPU-Features für schnellere Bildverarbeitung |
| RAM | 6144 MB | Immich empfiehlt ≥ 6 GB wegen Machine Learning |
| Netzwerk | VirtIO, Bridge `vmbr0` | Schnellste virtuelle Netzwerkkarte |

> Die Disk lässt sich später vergrössern, aber nur mühsam verkleinern.

## 2. Debian installieren

- Sprache Englisch (Fehlermeldungen lassen sich besser recherchieren), Tastatur Swiss German
- **Root-Passwort leer lassen** → der eigene Benutzer bekommt automatisch `sudo`, ein direkter root-Login ist nicht möglich
- Software: **nur „SSH server" + „standard system utilities"**, kein Desktop

**Warum kein Desktop?** Spart 1–2 GB RAM und Speicher, vor allem aber gilt: **weniger Software = weniger Angriffsfläche**.

## 3. Feste IP

```bash
ip a        # Name der Netzwerkkarte herausfinden, hier: ens18
sudo nano /etc/network/interfaces
```

```text
allow-hotplug ens18
iface ens18 inet static
    address <IMMICH-IP>/24
    gateway <ROUTER-IP>
```

### DNS-Problem nach der Umstellung

Nach dem Neustart ging `ping 1.1.1.1`, aber `ping deb.debian.org` meldete *„Temporary failure in name resolution"*.

**Ursache:** Mit DHCP bekommt die VM nicht nur die IP, sondern auch den DNS-Server mitgeteilt. Mit fester IP fällt das weg.

**Lösung:**
```bash
echo "nameserver <ROUTER-IP>" | sudo tee /etc/resolv.conf
```

> **Diagnose-Trick:** Zuerst per IP pingen (`1.1.1.1`), dann per Name. Geht nur die IP, ist es ein DNS-Problem und kein Netzwerkproblem.

Ab hier wurde per **SSH** vom Laptop gearbeitet, weil Kopieren und Einfügen in der Proxmox-Konsole nicht funktioniert.

## 4. Docker installieren

Über das **offizielle Docker-Repository** (nicht das ältere Debian-Paket):

```bash
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo docker run hello-world
```

**Warum der Schlüssel?** apt prüft damit die **Signatur** jedes Docker-Pakets. Manipulierte Pakete würden abgelehnt.

## 5. Immich installieren

Nach der offiziellen Anleitung: zwei Dateien aus den **offiziellen GitHub-Releases** laden.

```bash
mkdir ~/immich-app && cd ~/immich-app
curl -L -o docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
curl -L -o .env https://github.com/immich-app/immich/releases/latest/download/example.env
```

- `docker-compose.yml` = Bauplan: welche Container (Server, Machine Learning, Datenbank, Cache)
- `.env` = Einstellungen

In `.env` angepasst:
- `DB_PASSWORD` → zufälliges Passwort statt `postgres`:
  ```bash
  openssl rand -hex 16
  ```
- `TZ=Europe/Zurich` (Kommentarzeichen `#` entfernt)

Starten:
```bash
sudo docker compose up -d
```

Die Images kommen aus der **GitHub Container Registry** (`ghcr.io`) und vom Docker Hub. Sie sind in der Compose-Datei mit einem **sha256-Digest** festgelegt, Docker prüft also beim Download, dass genau dieses Image ankommt.

Weboberfläche: `http://<IMMICH-IP>:2283`. Der **erste** Account wird automatisch Admin.

## 6. Einstellungen beim Onboarding

| Einstellung | Wahl | Warum |
|---|---|---|
| Versionsprüfung | an | Hinweis auf (Sicherheits-)Updates, verschickt keine Fotodaten |
| Karte | optional | Kartenkacheln kommen von einem externen Dienst |
| Speichervorlage | **`YYYY/MM/DD`** | Fotos liegen auf der Disk in Datumsordnern und sind **auch ohne Immich** lesbar, wichtig für Backups |

## 7. Familie & Apps

- Jeder bekommt einen **eigenen Account** (*Administration → Benutzer*). Bibliotheken sind privat, auch der Admin sieht sie nicht automatisch.
- Teilen über **Partner-Freigabe** (ganze Bibliothek) oder **geteilte Alben**.
- Mobile App „Immich" → Server-Adresse eintragen → Backup-Alben wählen.
- Die App speichert Fotos **nicht doppelt**, nur App und Vorschaubild-Cache.

## Warum nur HTTP?

Immich liefert absichtlich nur HTTP und erwartet einen **Reverse Proxy** davor, der HTTPS übernimmt. Über Tailscale ist die Verbindung trotzdem verschlüsselt. Im Heim-WLAN nicht. HTTPS steht auf der [Roadmap](../README.md#roadmap).

## Offen

- [ ] Fotos aus OneDrive importieren
- [ ] Offline-Kopie auf externer HDD + Proxmox-Backups (3-2-1-Regel: 3 Kopien, 2 Medien, 1 an anderem Ort)

---

[← 03 – Proxmox VE](03-proxmox.md) · [Weiter: 05 – Pi-hole →](05-pihole.md)
