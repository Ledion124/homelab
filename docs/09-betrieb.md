# 09 – Betrieb

[← Zurück zur Übersicht](../README.md)

## Tagesablauf

Der Server läuft **tagsüber** und wird **nachts ausgeschaltet**, weil er im Wohnbereich steht.

### Einschalten
- Power-Knopf am Mini-PC drücken
- Proxmox startet von selbst, kein Monitor nötig. Nach 1–2 Minuten ist die Weboberfläche erreichbar
- Dank **Start at boot** starten Pi-hole → Immich → Claude-VM automatisch in dieser Reihenfolge
- Claude Code Remote Control muss (noch) von Hand gestartet werden:
  ```bash
  ssh <user>@<CLAUDE-IP>
  tmux new -s claude
  cd ~/projects && claude remote-control
  ```

### Ausschalten
- Weboberfläche: *pve → Shutdown*, oder Power-Knopf **kurz** drücken
- Proxmox fährt erst die VMs und Container sauber herunter (Qemu Agent), dann sich selbst
- **Nie** den Stecker ziehen und den Knopf nicht lange gedrückt halten, sonst können VMs und Dateien beschädigt werden
- Am Handy den WLAN-DNS auf *Automatisch* stellen, sonst kein Internet ohne Pi-hole

## Updates

| Wo | Befehl |
|---|---|
| Proxmox-Host | `apt update && apt full-upgrade` |
| Debian-VMs | `sudo apt update && sudo apt upgrade -y` |
| Pi-hole | `pihole -up`, Blocklisten: `pihole -g` |
| Immich | `cd ~/immich-app && sudo docker compose pull && sudo docker compose up -d` |
| Claude Code | aktualisiert sich selbst (`claude update` manuell) |

## Zugriff

| Wo | Wie |
|---|---|
| Zuhause | direkt über die IP |
| Unterwegs | Tailscale an, dann gleiche Adressen |
| Terminal vom Laptop | `ssh` in PowerShell |
| Terminal vom Handy | SSH-App (z. B. Termius) |
| Proxmox vom Handy | Browser, `https://<PVE-IP>:8006` |
| Claude vom Handy | Claude-App → Sitzung der VM |

## Wiederherstellung

- **Snapshots** vor riskanten Änderungen (*VM → Snapshots → Take Snapshot*)
- Backups auf externe HDD: geplant

---

[← 08 – Firewall](08-firewall.md) · [Weiter: 10 – Lessons Learned →](10-lessons-learned.md)
