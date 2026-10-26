# 07 – Claude-Code-VM

[← Zurück zur Übersicht](../README.md)

## Ziel

Eine eigene VM, in der **Claude Code** als KI-Agent an Projekten arbeitet, auch wenn mein Laptop zu ist. Gesteuert wird sie über die **Claude-App am Handy** oder den Browser.

**Remote Control** von Claude Code macht genau das: Die Sitzung läuft in der VM und arbeitet mit deren Dateien. Handy und Browser sind nur das „Fenster" dazu. Laut offizieller Doku baut die VM dabei **nur ausgehende HTTPS-Verbindungen** zur Anthropic API auf und öffnet **keine Ports**. Tailscale oder eine SSH-App braucht es dafür nicht, nur für Wartung.

**Einschränkung:** Die VM muss laufen. Ist der Server nachts aus, ist auch die Sitzung weg.

## Warum eine VM und kein Container?

Claude Code führt selbstständig Befehle aus. Eine VM hat einen **eigenen Kernel** und ist **stärker abgeschottet** als ein Container. Was in der VM passiert, bleibt in der VM. Ausserdem bekommt die VM **keine Zugangsdaten** zu Proxmox oder den anderen Diensten.

## 1. VM erstellen

| Einstellung | Wert | Warum |
|---|---|---|
| ID / Name | 102 / `claude` | |
| RAM | 4096 MB | Mindestanforderung von Claude Code |
| CPU | 2 Kerne, `host` | Die KI selbst läuft bei Anthropic, die VM führt nur Befehle aus |
| Disk | 32 GB, Discard | Platz für Projekte |
| Qemu Agent | an | |

Debian 13 wie bei [Immich](04-immich.md#2-debian-installieren): kein root-Passwort, nur SSH + Standard-Tools, kein Desktop.

## 2. Netzwerk

Feste IP per `sed` statt nano, um Tippfehler zu vermeiden:

```bash
sudo sed -i 's|iface ens18 inet dhcp|iface ens18 inet static\n    address <CLAUDE-IP>/24\n    gateway <ROUTER-IP>|' /etc/network/interfaces
echo "nameserver <ROUTER-IP>" | sudo tee /etc/resolv.conf
sudo reboot
```

**Warum der Router als DNS und nicht Pi-hole?** Claude Code braucht zuverlässig Internet. Wenn Pi-hole hängt oder etwas Wichtiges blockt, soll die VM trotzdem arbeiten können.

> Die Proxmox-Konsole unterstützt kein Einfügen. Darum zuerst mit `hostname -I` die DHCP-Adresse ablesen und per SSH weiterarbeiten. Achtung: Die Adresse nach `brd` in `ip a` ist die **Broadcast-Adresse**, nicht die eigene IP.

## 3. Werkzeuge und Claude Code

```bash
sudo apt update && sudo apt upgrade -y && sudo apt install -y tmux git curl

curl -fsSL https://claude.ai/install.sh -o claude-install.sh
less claude-install.sh
bash claude-install.sh          # bewusst OHNE sudo
exit                            # neu einloggen, damit ~/.local/bin im PATH ist
claude --version
```

Was beim Lesen des Installers auffiel:
- `set -e`: bricht beim ersten Fehler ab
- Die optionale Versionsangabe wird per **Regex validiert**
- Das Skript **verweigert `sudo`**, weil Claude Code in den Benutzerordner installiert wird und keine Root-Rechte braucht → **Least Privilege**

Der offizielle Installer hält Claude Code **automatisch aktuell**.

## 4. Login

```bash
mkdir -p ~/projects && cd ~/projects
claude
```

Login mit dem Claude-Abo (Pro/Max nötig) über einen Link im Browser. Die Frage *„Do you trust this folder?"* schützt davor, dass fremde Ordner mit versteckten Konfigurationsdateien automatisch Befehle auslösen. Bei einem selbst angelegten, leeren Ordner ist sie unbedenklich.

## 5. Snapshot

*102 → Snapshots → Take Snapshot*. Ein frischer, sauberer Stand, auf den sich die VM jederzeit zurücksetzen lässt, egal ob durch Claude, durch mich oder ein Update etwas kaputtgeht.

## 6. Remote Control in tmux

**tmux** hält Sitzungen am Leben, auch wenn die SSH-Verbindung getrennt wird.

```bash
tmux new -s claude
cd ~/projects
claude remote-control           # "Enable Remote Control?" → y
```

- **Leertaste** zeigt einen QR-Code → mit dem Handy scannen → Sitzung öffnet sich in der Claude-App
- **Abkoppeln, ohne zu beenden:** `Ctrl + B`, dann `D`
- **Wieder anhängen:** `tmux attach -t claude`
- **Nicht** `Ctrl + C`, das stoppt Remote Control

Test: In der App eine Datei anlegen lassen und im Terminal mit `cat` prüfen.

## 7. CLAUDE.md

Claude Code liest `CLAUDE.md` im Projektordner bei **jeder** Sitzung. Darin stehen Umgebung und Regeln:

```markdown
# Umgebung
- Du laeufst in einer Debian-13-VM (Hostname: claude) auf meinem Proxmox-Homelab.
- Das ist KEIN Cloud-Container. Dateien hier bleiben dauerhaft erhalten.
- Ich steuere dich ueber Claude Code Remote Control, oft vom Handy aus.
- Der Server laeuft nur tagsueber.

# Regeln
- Arbeite nur im Ordner ~/projects.
- Frag immer nach, bevor du Dateien loeschst oder etwas mit sudo ausfuehrst.
- Speichere niemals Passwoerter, Tokens oder Schluessel in Dateien.
- Greif nicht auf andere Geraete im Heimnetz zu.
```

> **Wichtig:** Das ist eine **Bitte**, keine technische Sperre. Die echte Grenze setzt die [Firewall](08-firewall.md).

## Was Claude in der VM kann und was nicht

| Kann | Kann nicht |
|---|---|
| Dateien im Benutzerordner lesen/ändern | Systemdateien ändern (`sudo` braucht ein Passwort, das Claude nicht kennt) |
| Befehle als normaler Benutzer ausführen | Die VM löschen (geht nur in Proxmox) |
| Ins Internet | Andere Geräte im Heimnetz erreichen (→ Firewall) |

Zusätzlich fragt Claude Code standardmässig vor Befehlen und Dateiänderungen nach. `--dangerously-skip-permissions` wird hier **nicht** verwendet.

## Offen

- [ ] Autostart von `claude remote-control` beim Booten (systemd)
- [ ] GitHub CLI mit einem Token, der nur auf einzelne Repos beschränkt ist

---

[← 06 – Tailscale](06-tailscale.md) · [Weiter: 08 – Firewall →](08-firewall.md)
